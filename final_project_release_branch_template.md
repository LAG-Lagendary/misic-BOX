Git Workflow for Final Release Version (The "Hammer" Deployment)

The goal is to move the production-ready code (the stable OMR pipeline and initial UI setup) from the development branch (develop) to a dedicated release branch, and finally merge it into main.

Step 1: Create the Release Branch

Create a branch for the first major version (e.g., v1.0.0) from the stable development branch.

# 1. Ensure you are on the development branch
git checkout develop

# 2. Pull the latest changes
git pull origin develop

# 3. Create the release branch
# Use "release/" prefix for consistency
git checkout -b release/v1.0.0


Step 2: Final Plan and "Hammer" Implementation Check

On the release/v1.0.0 branch:

Final Review: Perform final checks on the draft_plans_en.md and omr_pipeline.py files.

Version Bump: If you have version files (e.g., setup.py), update the version number to 1.0.0.

Documentation: Ensure all instructions (instruction_en.md) are up-to-date.

# Example: Commit final edits to the plan and pipeline before release
git add draft_plans_en.md omr_pipeline.py instruction_en.md
git commit -m "chore(release): Final review and preparation for v1.0.0"


Step 3: Tag and Merge to Main

This step is where the "hammer" is officially launched. Merge the stable release branch into main and tag the commit.

# 1. Move to the main branch
git checkout main

# 2. Merge the release branch into main (using --no-ff to keep history clean)
git merge release/v1.0.0 --no-ff

# 3. Create a version tag
git tag -a v1.0.0 -m "Initial Release: Automated OMR Pipeline (The First Hammer)"

# 4. Push the main branch and the tag to the remote repository
git push origin main
git push origin v1.0.0


Step 4: Cleanup and Return to Development

Merge the release branch back into develop to ensure develop contains all the final fixes and version updates, then delete the release branch.

# 1. Merge into develop
git checkout develop
git merge release/v1.0.0

# 2. Push development changes
git push origin develop

# 3. Delete the local and remote release branch (optional but recommended)
git branch -d release/v1.0.0
git push origin --delete release/v1.0.0
