# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Architecture Overview

This is a personal blog built using the **Minimal Mistakes Jekyll theme** (version 4.27.3), deployed on GitHub Pages. The site is configured as a fork of the upstream theme repository with customizations for personal blogging.

### Key Directory Structure

- `_posts/` - Blog posts in YYYY-MM-DD-title.md format with required front matter
- `_pages/` - Static pages 
- `_includes/` - Jekyll includes/partials for theme components
- `_layouts/` - Page layout templates
- `_sass/` - Stylesheets (SCSS)
- `_data/` - YAML data files for site configuration
- `assets/` - Static assets (images, JS, CSS)
- `images/` - User-uploaded images for blog posts
- `docs/` - Upstream theme documentation (should not be modified)

### Theme Architecture

The site uses the Minimal Mistakes theme via forking rather than gem-based installation. This means:

- All theme files are directly editable in the repository
- Updates from upstream require manual merging
- Full control over styling and functionality
- GitHub Pages compatible without additional plugins

## Development Commands

### Local Development
```bash
# Install dependencies (theme development only)
bundle install

# Preview theme changes (uses test/ directory)
bundle exec rake preview

# Build JavaScript assets
bundle exec rake js

# Update copyright headers and version info
bundle exec rake copyright
```

### Jekyll Post Requirements

Blog posts MUST follow these conventions:
1. **File naming**: `YYYY-MM-DD-title.md` format in `_posts/` directory
2. **Front matter**: Required fields:
   ```yaml
   ---
   layout: single
   title: "Post Title"
   date: YYYY-MM-DD
   categories: CategoryName
   ---
   ```

### Configuration

- `_config.yml` - Main Jekyll configuration
  - Site uses "dirt" skin theme
  - Disqus comments enabled ("JoonHyoungLee-seoul")
  - Google Analytics configured
  - Shanghai timezone
  - Author profile enabled

### Important Notes

- **Posts directory**: Must use `_posts/` (not `posts/`) for Jekyll recognition
- **GitHub Pages**: Site auto-deploys from master branch pushes
- **Remote tracking**: Upstream set to original mmistakes/minimal-mistakes for theme updates
- **Custom modifications**: Check git history for user customizations before reverting changes

### Common Issues

1. **Posts not appearing**: Verify `_posts/` directory name and front matter `date` field
2. **Theme updates**: Use `git fetch upstream` and merge carefully to preserve customizations
3. **Build failures**: Check that `jekyll-include-cache` plugin is maintained in `_config.yml` plugins array

### Site Information

- **URL**: https://joonhyounglee-seoul.github.io/
- **Author**: JoonHyoung Lee
- **Location**: Shanghai
- **Email**: ljh012244@gmail.com
- **Categories**: Currently focused on Docker tutorials