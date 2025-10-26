# -*- coding: utf-8 -*-
import fitz # PyMuPDF
import os
import re
import shutil
import subprocess
import argparse
from pdf2image import convert_from_path
import cv2
import numpy as np
import pytesseract

# --- CONSTANTS AND SETTINGS ---
TEMP_DIR = "temp_omr_files"
DPI = 300 # Recommended resolution for OMR

# --- STEP 1: Split PDF into Images and Individual PDF Pages ---
def split_pdf_to_images_and_pages(pdf_path, temp_output_dir):
    """
    Splits the source PDF into individual PDF pages and PNG images.

    :param pdf_path: Path to the source PDF file.
    :param temp_output_dir: Temporary directory for saving files.
    :return: List of paths to the generated PNG images.
    """
    print(f"[{'STEP 1':<6}] Splitting PDF into pages ({DPI} dpi)...")

    # 1. Create the temporary directory
    os.makedirs(temp_output_dir, exist_ok=True)

    image_paths = []

    try:
        # 2. Save each page as a separate PDF (for folder 1)
        doc = fitz.open(pdf_path)
        print(f"Pages detected: {len(doc)}")

        for i, page in enumerate(doc):
            page_num = i + 1

            # Saving as a separate PDF file
            single_page_pdf_path = os.path.join(temp_output_dir, f"page_{page_num}.pdf")
            single_page = fitz.open()
            single_page.insert_pdf(doc, from_page=i, to_page=i)
            single_page.save(single_page_pdf_path)
            single_page.close()

        # 3. Convert to images for Audiveris processing
        images = convert_from_path(pdf_path, dpi=DPI)
        for i, img in enumerate(images):
            page_num = i + 1
            img_path = os.path.join(temp_output_dir, f"img_page_{page_num}.png")
            img.save(img_path)
            image_paths.append(img_path)

    except Exception as e:
        print(f"Error in Step 1: {e}")
        return []

    print(f"[{'STEP 1':<6}] Completed. Created {len(image_paths)} images.")
    return image_paths

# --- STEP 2: Analyze Structure and Identify Pieces ---
def detect_piece_boundaries(image_paths):
    """
    Analyzes images to determine the start of new musical pieces
    by looking for large text blocks (titles).

    :param image_paths: List of paths to page images.
    :return: List of dictionaries with piece boundaries:
             [{'title': 'Title', 'pages': [1, 2, ...]}, ...]
    """
    print(f"[{'STEP 2':<6}] Structure analysis: searching for piece titles...")

    pieces = []
    current_title = "UNTITLED_PIECE_1"
    current_pages = []
    piece_counter = 1

    for i, path in enumerate(image_paths):
        page_num = i + 1

        # Load image for Tesseract
        img = cv2.imread(path)

        # Convert to grayscale and binarize for better recognition
        gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)

        # Text recognition settings (OSD)
        try:
            # Use configuration for text recognition (PSM 6: Assume a single uniform block of text)
            custom_config = r'--oem 3 --psm 6'
            # Recognize both Russian and English text
            text = pytesseract.image_to_string(gray, lang='rus+eng', config=custom_config)

            # Filtering and cleaning the text
            cleaned_text = re.sub(r'[\s\n]+', ' ', text).strip()

            # Check for a potential piece title:
            is_title = False
            if cleaned_text and len(cleaned_text) > 5 and re.match(r'^[A-ZА-Я0-9]', cleaned_text):
                 # Simple heuristic: if the text is large enough (e.g., occupies the top part of the page)
                 # Look for text blocks in the upper part of the page (DPI/2)
                data = pytesseract.image_to_data(gray, lang='rus+eng', config=custom_config, output_type=pytesseract.Output.DICT)

                top_margin = DPI // 2 # ~1.5 inches from the top

                for k in range(len(data['text'])):
                    # Confidence above 70% AND large block at the top
                    if int(data['conf'][k]) > 70 and data['top'][k] < top_margin and data['height'][k] > 20:
                        is_title = True
                        # Assign the title, cleaned of unnecessary characters
                        current_title = cleaned_text.replace('/', '_').replace(':', ' -')
                        break

            if is_title and page_num not in current_pages:
                if current_pages:
                    # Save the previous piece
                    pieces.append({'title': current_title, 'pages': current_pages})

                # Update the title and start a new piece
                if not current_title:
                    piece_counter += 1
                    current_title = f"UNTITLED_PIECE_{piece_counter}"

                current_pages = [page_num]

            else:
                current_pages.append(page_num)

        except pytesseract.TesseractNotFoundError:
            print("WARNING: Tesseract not found. Skipping Step 2 or install Tesseract.")
            current_pages.append(page_num) # Just add the page

        except Exception as e:
            print(f"Error recognizing text on page {page_num}: {e}")
            current_pages.append(page_num)

    if current_pages:
        # Add the last piece if it wasn't added
        pieces.append({'title': current_title, 'pages': current_pages})

    print(f"[{'STEP 2':<6}] Detected {len(pieces)} pieces/sections.")
    return pieces

# --- STEP 3: Create Folder 1 (PDF Pieces with Markup) ---
def create_piece_pdfs(pieces, input_pdf_path, output_dir, temp_dir):
    """
    Assembles pages belonging to one piece into a single PDF file.

    :param pieces: List of pieces with page numbers.
    :param input_pdf_path: Path to the source PDF.
    :param output_dir: Target directory for saving PDF pieces.
    :param temp_dir: Temporary directory where individual pages are located.
    """
    print(f"[{'STEP 3':<6}] Assembling individual PDF pieces in: {output_dir}")

    pdf_output_dir = os.path.join(output_dir, "pdf_pieces")
    os.makedirs(pdf_output_dir, exist_ok=True)

    # Open the source document only once
    original_doc = fitz.open(input_pdf_path)

    for piece in pieces:
        # Sanitize title for filename
        piece_title = piece['title'].replace('/', '_').replace('\\', '_')
        piece_doc = fitz.open()

        for page_num in piece['pages']:
            # Insert the page from the original document
            piece_doc.insert_pdf(original_doc, from_page=page_num-1, to_page=page_num-1)

        # OPTIONAL: Add simple markup to the first page
        if piece_doc:
             first_page = piece_doc[0]
             # Markup with a red border (for visualizing boundaries)
             first_page.draw_rect(fitz.Rect(20, 20, first_page.rect.width - 20, first_page.rect.height - 20),
                                  color=(1, 0, 0), width=3)

        # Save the assembled PDF
        output_path = os.path.join(pdf_output_dir, f"{piece_title}.pdf")
        piece_doc.save(output_path)
        piece_doc.close()

    original_doc.close()
    print(f"[{'STEP 3':<6}] PDF piece assembly completed.")

# --- STEP 4: Audiveris Execution Emulation/Run ---
def run_audiveris_emulation(image_paths, temp_dir, audiveris_jar):
    """
    Runs Audiveris for OMR. (Requires installed Audiveris.jar and Java).

    :param image_paths: List of paths to page images.
    :param temp_dir: Temporary directory for intermediate results.
    :param audiveris_jar: Path to the Audiveris.jar file.
    :return: Names of directories created by Audiveris.
    """
    print(f"[{'STEP 4':<6}] Starting Audiveris (OPTICAL MUSIC RECOGNITION)...")

    if not os.path.exists(audiveris_jar):
        print("!!! WARNING: Audiveris JAR not found. !!!")
        print(f"File: {audiveris_jar} does not exist. This step will be skipped.")

        # Create empty XML files for emulation
        emulated_dirs = []
        for path in image_paths:
             # Filename without extension
            base_name = os.path.basename(path).split('.')[0]
            emulated_dir = os.path.join(temp_dir, f"{base_name}-OMR")
            os.makedirs(emulated_dir, exist_ok=True)

            # Create an empty MusicXML file
            with open(os.path.join(emulated_dir, f"{base_name}.mxl"), 'w') as f:
                 f.write(f"<!-- MusicXML emulation for {base_name} -->")
            emulated_dirs.append(emulated_dir)

        return emulated_dirs

    # --- Actual Audiveris execution (if JAR is found) ---
    audiveris_output_dirs = []

    for path in image_paths:
        try:
            cmd = [
                "java", "-jar", audiveris_jar,
                "-in", path,
                "-out", temp_dir,
                "-batch"  # GUI-less mode to save resources
            ]
            print(f"Processing: {os.path.basename(path)}")
            # Run the process. check=True will raise an error if the command fails.
            subprocess.run(cmd, check=True, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)

            # Audiveris creates a folder in temp_dir with the -OMR suffix
            base_name = os.path.basename(path).split('.')[0]
            omr_dir = os.path.join(temp_dir, f"{base_name}-OMR")
            audiveris_output_dirs.append(omr_dir)

        except subprocess.CalledProcessError as e:
            print(f"Error executing Audiveris for {path}: {e}")
        except FileNotFoundError:
            print("Error: 'java' command not found. Ensure Java is installed and in PATH.")
            break

    print(f"[{'STEP 4':<6}] Audiveris execution completed.")
    return audiveris_output_dirs

# --- STEP 5: Create Folder 2 (MusicXML Files) ---
def collect_musicxml(audiveris_output_dirs, output_dir, pieces):
    """
    Collects all generated MusicXML files into one folder.

    :param audiveris_output_dirs: List of directories created by Audiveris.
    :param output_dir: Target directory.
    :param pieces: List of pieces for renaming (if needed).
    """
    print(f"[{'STEP 5':<6}] Assembling MusicXML in: {output_dir}")

    musicxml_output_dir = os.path.join(output_dir, "musicxml_files")
    os.makedirs(musicxml_output_dir, exist_ok=True)

    # Dictionary to track page -> piece title
    page_to_title = {}
    for piece in pieces:
        for page_num in piece['pages']:
            # Audiveris filename: img_page_N.mxl
            page_to_title[f"img_page_{page_num}"] = piece['title']

    xml_counter = 0

    for omr_dir in audiveris_output_dirs:
        # Base name of the file we are looking for: "img_page_N"
        base_name = os.path.basename(omr_dir).replace("-OMR", "")

        for file_name in os.listdir(omr_dir):
            if file_name.endswith((".xml", ".mxl")): # Audiveris may generate MXL
                source_path = os.path.join(omr_dir, file_name)

                # Attempt to rename using the piece title (assigned to the first page of the piece)
                piece_title = page_to_title.get(base_name, f"unknown_page_{base_name}")
                new_file_name = f"{piece_title}_{base_name}.{file_name.split('.')[-1]}"
                new_file_name = new_file_name.replace(' ', '_').replace('/', '_').replace('\\', '_')

                destination_path = os.path.join(musicxml_output_dir, new_file_name)

                shutil.copy(source_path, destination_path)
                xml_counter += 1

    print(f"[{'STEP 5':<6}] MusicXML assembly completed. Copied {xml_counter} files.")

# --- MAIN PIPELINE FUNCTION ---
def main():
    parser = argparse.ArgumentParser(description="Automated OMR Pipeline (PDF -> MusicXML) using Audiveris.")
    parser.add_argument("--input", required=True, help="Path to the source PDF file containing musical scores.")
    parser.add_argument("--output_dir", required=True, help="Target directory for results (pdf_pieces and musicxml_files).")
    parser.add_argument("--audiveris_jar", default="./Audiveris.jar", help="Path to the Audiveris.jar executable file (default: ./Audiveris.jar).")
    parser.add_argument("--clean", action="store_true", help="Remove temporary files after completion.")

    args = parser.parse_args()

    input_pdf_path = args.input
    output_dir = args.output_dir
    audiveris_jar = args.audiveris_jar

    if not os.path.exists(input_pdf_path):
        print(f"Error: Input file not found at path: {input_pdf_path}")
        return

    # Prepare directories
    temp_dir = os.path.join(output_dir, TEMP_DIR)
    os.makedirs(output_dir, exist_ok=True)

    all_image_paths = []
    piece_boundaries = []
    audiveris_output_dirs = []

    # 1. Split PDF into pages and images
    all_image_paths = split_pdf_to_images_and_pages(input_pdf_path, temp_dir)
    if not all_image_paths: return

    # 2. Analyze structure and identify pieces
    piece_boundaries = detect_piece_boundaries(all_image_paths)

    # 3. Create Folder 1 (PDF Pieces with Markup)
    create_piece_pdfs(piece_boundaries, input_pdf_path, output_dir, temp_dir)

    # 4. Audiveris Processing (Conversion to MusicXML)
    audiveris_output_dirs = run_audiveris_emulation(all_image_paths, temp_dir, audiveris_jar)

    # 5. Create Folder 2 (MusicXML Files)
    if audiveris_output_dirs:
        collect_musicxml(audiveris_output_dirs, output_dir, piece_boundaries)

    # Cleanup
    if args.clean and os.path.exists(temp_dir):
        print(f"[{'CLEANUP':<6}] Removing temporary directory: {temp_dir}")
        shutil.rmtree(temp_dir)
    elif os.path.exists(temp_dir):
        print(f"[{'TEMP':<6}] Temporary files saved in: {temp_dir}")

    print("\n[COMPLETED] OMR Pipeline execution finished.")

if __name__ == "__main__":
    # In a real environment, you might set the logging level for pytesseract/opencv
    # to avoid cluttering the console, but we keep it verbose for debugging here.
    main()
