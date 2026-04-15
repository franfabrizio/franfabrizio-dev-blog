# Remove the file from filesystem
rm content/testing-your-dkim-and-spf-setup/index.md

# Remove it from git tracking
git rm content/testing-your-dkim-and-spf-setup/index.md

# Commit the change
git commit -m "Remove duplicate index.md from wrong location"

# Restore missing files
git checkout content/testing-your-dkim-and-spf-setup/index.md
git submodule update --init --recursive
