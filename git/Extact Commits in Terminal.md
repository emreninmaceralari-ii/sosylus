#!/bin/bash

# Exit on any error
set -euo pipefail

# Configuration
COMMIT_COUNT=15
OUTPUT_DIR="docs"

# Create output directory
mkdir -p "$OUTPUT_DIR"

echo "Starting commit export..."
echo "Target: $COMMIT_COUNT commits"
echo "Output: $OUTPUT_DIR/"
echo ""

# Counter for verification
count=0

# Process commits
git log -${COMMIT_COUNT} --format="%ad|%h|%H|%s" --date=format:"%d-%m-%Y" | while IFS='|' read -r date hash fullhash msg; do
  count=$((count + 1))
  
  # Validate all fields are non-empty
  if [ -z "$date" ] || [ -z "$hash" ] || [ -z "$fullhash" ] || [ -z "$msg" ]; then
    echo "⚠️  Skipping malformed commit data (empty field)"
    continue
  fi
  
  # Sanitize message for filename (replace ALL unsafe chars)
  clean_msg=$(echo "${msg:0:20}" | sed 's/[^a-zA-Z0-9()_-]/_/g' | sed 's/__*/_/g' | sed 's/^_//;s/_$//')
  
  # Ensure clean_msg is not empty after sanitization
  if [ -z "$clean_msg" ]; then
    clean_msg="commit"
  fi
  
  # Build safe filename
  filename="${OUTPUT_DIR}/${date}-${hash}-${clean_msg}.md"
  
  echo "[$count/$COMMIT_COUNT] Processing: $hash - ${msg:0:50}..."
  
  # Generate markdown with error handling
  {
    echo "# Commit Details"
    echo ""
    echo "## Metadata"
    echo "- **Short Hash:** \`$hash\`"
    echo "- **Full Hash:** \`$fullhash\`"
    echo "- **Date:** $date"
    echo "- **Message:** $msg"
    echo ""
    
    echo "### Author Information"
    git show --format="- **Name:** %an%n- **Email:** %ae" -s "$fullhash" 2>/dev/null || echo "- **Error:** Unable to fetch author info"
    echo ""
    
    echo "### Committer Information"
    git show --format="- **Name:** %cn%n- **Email:** %ce" -s "$fullhash" 2>/dev/null || echo "- **Error:** Unable to fetch committer info"
    echo ""
    
    echo "### Parent Commits"
    parents=$(git show --format="%P" -s "$fullhash" 2>/dev/null || echo "")
    if [ -z "$parents" ]; then
      echo "- *(Initial commit - no parents)*"
    else
      for parent in $parents; do
        echo "- \`$parent\`"
      done
    fi
    echo ""
    echo "---"
    echo ""
    
    echo "## Files Changed"
    echo ""
    git diff-tree --no-commit-id --name-status -r "$fullhash" 2>/dev/null | while read status file; do
      if [ -n "$status" ] && [ -n "$file" ]; then
        echo "- **$file** (\`$status\`)"
        git diff-tree --no-commit-id --numstat -r "$fullhash" 2>/dev/null | grep -F "$file" | head -1 | while read add del fname; do
          echo "  - Additions: +$add"
          echo "  - Deletions: -$del"
        done
      fi
    done
    echo ""
    
    echo "### Change Summary Statistics"
    git show --shortstat --format="" "$fullhash" 2>/dev/null || echo "*(Statistics unavailable)*"
    echo ""
    echo "---"
    echo ""
    
    echo "## Full Commit Information"
    echo ""
    echo '```'
    git show --format=fuller --stat "$fullhash" 2>/dev/null || echo "Error: Unable to fetch commit details"
    echo '```'
    echo ""
    echo "---"
    echo ""
    
    echo "## Diff"
    echo ""
    echo '```diff'
    git show --format="" "$fullhash" 2>/dev/null || echo "Error: Unable to fetch diff"
    echo '```'
  } > "$filename" 2>&1
  
  if [ $? -eq 0 ]; then
    echo "  ✓ Created: $(basename "$filename")"
  else
    echo "  ✗ Failed: $(basename "$filename")"
  fi
done

echo ""
echo "=== Summary ==="
actual_count=$(ls -1 "$OUTPUT_DIR"/*.md 2>/dev/null | wc -l)
echo "Expected: $COMMIT_COUNT commits"
echo "Created: $actual_count files"

if [ "$actual_count" -eq "$COMMIT_COUNT" ]; then
  echo "✓ Success! All commits exported."
else
  echo "⚠️  Warning: Expected $COMMIT_COUNT but got $actual_count files."
fi