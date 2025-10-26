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

# --- КОНСТАНТЫ И НАСТРОЙКИ ---
TEMP_DIR = "temp_omr_files"
DPI = 300 # Рекомендуемое разрешение для OMR

# --- ШАГ 1: Разбиение PDF на изображения и отдельные страницы PDF ---
def split_pdf_to_images_and_pages(pdf_path, temp_output_dir):
    """
    Разбивает исходный PDF на отдельные PDF-страницы и PNG-изображения.

    :param pdf_path: Путь к исходному PDF-файлу.
    :param temp_output_dir: Временная директория для сохранения файлов.
    :return: Список путей к сгенерированным PNG-изображениям.
    """
    print(f"[{'ШАГ 1':<5}] Разбиение PDF на страницы ({DPI} dpi)...")

    # 1. Создаем временную директорию
    os.makedirs(temp_output_dir, exist_ok=True)

    image_paths = []

    try:
        # 2. Сохраняем каждую страницу как отдельный PDF (для папки 1)
        doc = fitz.open(pdf_path)
        print(f"Обнаружено страниц: {len(doc)}")

        for i, page in enumerate(doc):
            page_num = i + 1

            # Сохранение как отдельный PDF-файл
            single_page_pdf_path = os.path.join(temp_output_dir, f"page_{page_num}.pdf")
            single_page = fitz.open()
            single_page.insert_pdf(doc, from_page=i, to_page=i)
            single_page.save(single_page_pdf_path)
            single_page.close()

        # 3. Конвертируем в изображения для Audiveris
        images = convert_from_path(pdf_path, dpi=DPI)
        for i, img in enumerate(images):
            page_num = i + 1
            img_path = os.path.join(temp_output_dir, f"img_page_{page_num}.png")
            img.save(img_path)
            image_paths.append(img_path)

    except Exception as e:
        print(f"Ошибка на Шаге 1: {e}")
        return []

    print(f"[{'ШАГ 1':<5}] Завершено. Создано {len(image_paths)} изображений.")
    return image_paths

# --- ШАГ 2: Анализ структуры и выделение пьес ---
def detect_piece_boundaries(image_paths):
    """
    Анализирует изображения для определения начала новых музыкальных пьес
    по крупным текстовым блокам (названиям).

    :param image_paths: Список путей к изображениям страниц.
    :return: Список словарей с границами пьес:
             [{'title': 'Название', 'pages': [1, 2, ...]}, ...]
    """
    print(f"[{'ШАГ 2':<5}] Анализ структуры: поиск названий пьес...")

    pieces = []
    current_title = "UNTITLED_PIECE_1"
    current_pages = []
    piece_counter = 1

    for i, path in enumerate(image_paths):
        page_num = i + 1

        # Загрузка изображения для Tesseract
        img = cv2.imread(path)

        # Преобразование в оттенки серого и бинаризация для лучшего распознавания
        gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)

        # Настройка OSD (распознавание текста)
        # Tesseract лучше распознает печатный текст, чем ноты.
        try:
            # Используем конфигурацию для распознавания текста (PSM 6: Assume a single uniform block of text)
            custom_config = r'--oem 3 --psm 6'
            text = pytesseract.image_to_string(gray, lang='rus+eng', config=custom_config)

            # Фильтрация и очистка текста
            cleaned_text = re.sub(r'[\s\n]+', ' ', text).strip()

            # Проверка на потенциальное название пьесы:
            # - Должно быть достаточно длинным (минимум 5 символов)
            # - Должно начинаться с заглавной буквы или цифры
            # - Не должно содержать много знаков препинания
            is_title = False
            if cleaned_text and len(cleaned_text) > 5 and re.match(r'^[A-ZА-Я0-9]', cleaned_text):
                 # Простая эвристика: если текст достаточно крупный (например, занимает верхнюю часть страницы)
                 # Ищем текстовые блоки в верхней части страницы (DPI/2)
                data = pytesseract.image_to_data(gray, lang='rus+eng', config=custom_config, output_type=pytesseract.Output.DICT)

                top_margin = DPI // 2 # 1.5 дюйма от верха

                for k in range(len(data['text'])):
                    if int(data['conf'][k]) > 70: # Уверенность выше 70%
                        if data['top'][k] < top_margin and data['height'][k] > 20: # Крупный блок вверху
                            is_title = True
                            # Присваиваем название, очищенное от лишних символов
                            current_title = cleaned_text.replace('/', '_').replace(':', ' -')
                            break

            if is_title and page_num not in current_pages:
                if current_pages:
                    pieces.append({'title': current_title, 'pages': current_pages})

                # Обновляем название и начинаем новую пьесу
                # Если название повторяется или не найдено, используем счетчик
                if not current_title:
                    piece_counter += 1
                    current_title = f"UNTITLED_PIECE_{piece_counter}"

                current_pages = [page_num]

            else:
                current_pages.append(page_num)

        except pytesseract.TesseractNotFoundError:
            print("ВНИМАНИЕ: Tesseract не найден. Пропустите Шаг 2 или установите Tesseract.")
            current_pages.append(page_num) # Просто добавляем страницу

        except Exception as e:
            print(f"Ошибка при распознавании текста на странице {page_num}: {e}")
            current_pages.append(page_num)

    if current_pages:
        # Если последняя пьеса не была добавлена
        pieces.append({'title': current_title, 'pages': current_pages})

    print(f"[{'ШАГ 2':<5}] Обнаружено {len(pieces)} пьес/разделов.")
    return pieces

# --- ШАГ 3: Формирование папки 1 (PDF-пьесы с разметкой) ---
def create_piece_pdfs(pieces, input_pdf_path, output_dir, temp_dir):
    """
    Собирает страницы, принадлежащие одной пьесе, в один PDF-файл.

    :param pieces: Список пьес с номерами страниц.
    :param input_pdf_path: Путь к исходному PDF.
    :param output_dir: Целевая директория для сохранения PDF-пьес.
    :param temp_dir: Временная директория, где лежат отдельные страницы.
    """
    print(f"[{'ШАГ 3':<5}] Сборка отдельных PDF-пьес в: {output_dir}")

    pdf_output_dir = os.path.join(output_dir, "pdf_pieces")
    os.makedirs(pdf_output_dir, exist_ok=True)

    # Открываем исходный документ только один раз
    original_doc = fitz.open(input_pdf_path)

    for piece in pieces:
        piece_title = piece['title'].replace('/', '_').replace('\\', '_')
        piece_doc = fitz.open()

        for page_num in piece['pages']:
            # Вставляем страницу из исходного документа
            piece_doc.insert_pdf(original_doc, from_page=page_num-1, to_page=page_num-1)

        # ОПЦИОНАЛЬНО: Добавление простой разметки на первую страницу
        # Для демонстрации, рисуем простую рамку
        if piece_doc:
             first_page = piece_doc[0]
             # Разметка красной рамкой (для визуализации границ)
             first_page.draw_rect(fitz.Rect(20, 20, first_page.rect.width - 20, first_page.rect.height - 20),
                                  color=(1, 0, 0), width=3)

        # Сохранение собранного PDF
        output_path = os.path.join(pdf_output_dir, f"{piece_title}.pdf")
        piece_doc.save(output_path)
        piece_doc.close()

    original_doc.close()
    print(f"[{'ШАГ 3':<5}] Сборка PDF-пьес завершена.")

# --- ШАГ 4: Эмуляция вызова Audiveris ---
def run_audiveris_emulation(image_paths, temp_dir, audiveris_jar):
    """
    Запуск Audiveris для OMR. (Требуется установленный Audiveris.jar и Java).

    :param image_paths: Список путей к изображениям страниц.
    :param temp_dir: Временная директория для сохранения промежуточных результатов.
    :param audiveris_jar: Путь к файлу Audiveris.jar.
    :return: Имена директорий, созданных Audiveris.
    """
    print(f"[{'ШАГ 4':<5}] Запуск Audiveris (ОПТИЧЕСКОЕ РАСПОЗНАВАНИЕ НОТ)...")

    if not os.path.exists(audiveris_jar):
        print("!!! ПРЕДУПРЕЖДЕНИЕ: Audiveris JAR не найден. !!!")
        print(f"Файл: {audiveris_jar} не существует. Этот шаг будет пропущен.")
        print("Для запуска, пожалуйста, скачайте Audiveris.jar и укажите верный путь.")

        # Создаем пустые XML-файлы для эмуляции работы
        emulated_dirs = []
        for path in image_paths:
             # Имя файла без расширения
            base_name = os.path.basename(path).split('.')[0]
            emulated_dir = os.path.join(temp_dir, f"{base_name}-OMR")
            os.makedirs(emulated_dir, exist_ok=True)

            # Создаем пустой MusicXML файл
            with open(os.path.join(emulated_dir, f"{base_name}.mxl"), 'w') as f:
                 f.write(f"<!-- MusicXML эмуляция для {base_name} -->")
            emulated_dirs.append(emulated_dir)

        return emulated_dirs

    # --- Реальный вызов Audiveris (если JAR найден) ---
    audiveris_output_dirs = []

    for path in image_paths:
        try:
            cmd = [
                "java", "-jar", audiveris_jar,
                "-in", path,
                "-out", temp_dir,
                "-batch"  # Режим без GUI
            ]
            print(f"Обработка: {os.path.basename(path)}")
            # Запуск процесса. check=True вызовет ошибку, если команда завершится неудачей.
            subprocess.run(cmd, check=True, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)

            # Audiveris создаёт папку в temp_dir с суффиксом -OMR
            base_name = os.path.basename(path).split('.')[0]
            omr_dir = os.path.join(temp_dir, f"{base_name}-OMR")
            audiveris_output_dirs.append(omr_dir)

        except subprocess.CalledProcessError as e:
            print(f"Ошибка выполнения Audiveris для {path}: {e}")
        except FileNotFoundError:
            print("Ошибка: Команда 'java' не найдена. Убедитесь, что Java установлена и доступна в PATH.")
            break

    print(f"[{'ШАГ 4':<5}] Выполнение Audiveris завершено.")
    return audiveris_output_dirs

# --- ШАГ 5: Формирование папки 2 (MusicXML-файлы) ---
def collect_musicxml(audiveris_output_dirs, output_dir, pieces):
    """
    Собирает все сгенерированные MusicXML-файлы в одну папку.

    :param audiveris_output_dirs: Список директорий, созданных Audiveris.
    :param output_dir: Целевая директория.
    :param pieces: Список пьес для переименования (если нужно).
    """
    print(f"[{'ШАГ 5':<5}] Сборка MusicXML в: {output_dir}")

    musicxml_output_dir = os.path.join(output_dir, "musicxml_files")
    os.makedirs(musicxml_output_dir, exist_ok=True)

    # Словарь для отслеживания страницы -> название пьесы
    page_to_title = {}
    for piece in pieces:
        for page_num in piece['pages']:
            # Имя файла Audiveris: img_page_N.mxl
            page_to_title[f"img_page_{page_num}"] = piece['title']

    xml_counter = 0

    for omr_dir in audiveris_output_dirs:
        # Имя файла, который ищем: "img_page_N.mxl" (Audiveris)
        base_name = os.path.basename(omr_dir).replace("-OMR", "")

        for file_name in os.listdir(omr_dir):
            if file_name.endswith((".xml", ".mxl")): # Audiveris может генерировать MXL
                source_path = os.path.join(omr_dir, file_name)

                # Попытка переименования по названию пьесы (присваиваем название первой странице пьесы)
                piece_title = page_to_title.get(base_name, f"unknown_page_{base_name}")
                new_file_name = f"{piece_title}_{base_name}.{file_name.split('.')[-1]}"
                new_file_name = new_file_name.replace(' ', '_').replace('/', '_').replace('\\', '_')

                destination_path = os.path.join(musicxml_output_dir, new_file_name)

                shutil.copy(source_path, destination_path)
                xml_counter += 1

    print(f"[{'ШАГ 5':<5}] Сборка MusicXML завершена. Скопировано {xml_counter} файлов.")

# --- ГЛАВНАЯ ФУНКЦИЯ ПАЙПЛАЙНА ---
def main():
    parser = argparse.ArgumentParser(description="Автоматический пайплайн OMR (PDF -> MusicXML) с использованием Audiveris.")
    parser.add_argument("--input", required=True, help="Путь к исходному PDF-файлу со сборником нот.")
    parser.add_argument("--output_dir", required=True, help="Целевая директория для результатов (pdf_pieces и musicxml_files).")
    parser.add_argument("--audiveris_jar", default="./Audiveris.jar", help="Путь к исполняемому файлу Audiveris.jar (по умолчанию: ./Audiveris.jar).")
    parser.add_argument("--clean", action="store_true", help="Удалить временные файлы после завершения.")

    args = parser.parse_args()

    input_pdf_path = args.input
    output_dir = args.output_dir
    audiveris_jar = args.audiveris_jar

    if not os.path.exists(input_pdf_path):
        print(f"Ошибка: Входной файл не найден по пути: {input_pdf_path}")
        return

    # Подготовка директорий
    temp_dir = os.path.join(output_dir, TEMP_DIR)
    os.makedirs(output_dir, exist_ok=True)

    all_image_paths = []
    piece_boundaries = []
    audiveris_output_dirs = []

    # 1. Разбиение PDF на страницы и изображения
    all_image_paths = split_pdf_to_images_and_pages(input_pdf_path, temp_dir)
    if not all_image_paths: return

    # 2. Анализ структуры и выделение пьес
    piece_boundaries = detect_piece_boundaries(all_image_paths)

    # 3. Формирование папки 1 (PDF-пьесы с разметкой)
    create_piece_pdfs(piece_boundaries, input_pdf_path, output_dir, temp_dir)

    # 4. Обработка Audiveris (конверсия в MusicXML)
    audiveris_output_dirs = run_audiveris_emulation(all_image_paths, temp_dir, audiveris_jar)

    # 5. Формирование папки 2 (MusicXML-файлы)
    if audiveris_output_dirs:
        collect_musicxml(audiveris_output_dirs, output_dir, piece_boundaries)

    # Очистка
    if args.clean and os.path.exists(temp_dir):
        print(f"[{'ОЧИСТКА':<5}] Удаление временной директории: {temp_dir}")
        shutil.rmtree(temp_dir)
    elif os.path.exists(temp_dir):
        print(f"[{'ВРЕМЕННЫЕ':<5}] Временные файлы сохранены в: {temp_dir}")

    print("\n[ЗАВЕРШЕНО] Пайплайн OMR выполнен.")

if __name__ == "__main__":
    # Установка уровня логирования для pytesseract/opencv, чтобы не засорять консоль
    # (Требуется для production-кода, здесь оставляем для отладки)
    main()
