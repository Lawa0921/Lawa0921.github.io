# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a Jekyll-based website for "密式旅行" (MissTravel), a camping and lodging facility in Miaoli, Taiwan. The site showcases accommodations, provides booking information, and serves as a business portal for guests.

## Development Commands

### Prerequisites
```bash
# Install Ruby and Bundler first, then:
bundle install
```

### Local Development
```bash
# Serve the site locally with auto-reload
bundle exec jekyll serve

# Build the site without serving
bundle exec jekyll build

# Build for production
JEKYLL_ENV=production bundle exec jekyll build
```

### Deployment
The site automatically deploys to GitHub Pages when pushing to the main branch. The custom domain is configured via the CNAME file pointing to `misstravel.me`.

## Architecture Overview

### Jekyll Collections
The site uses Jekyll collections for structured content:

- **_rooms/**: Accommodation listings (campsites, log cabins, suites)
  - Each room has `weekday_price`, `holiday_price`, and `standard_price` fields
  - Images are stored in `assets/images/rooms/[room-name]/`
  
- **_infos/**: Information pages (accounts, maps, rules, menus, contact)
  - Special URLs configured in `_config.yml` for each info type
  
- **_announcements/**: Site announcements and updates

### Key Technologies
- **Jekyll 4.2**: Static site generator
- **Bootstrap 4**: Full SCSS source included in `_sass/bootstrap/`
- **HTML5 UP "Forty" Theme**: Base theme with heavy customizations
- **Formspree.io**: Contact form integration (email configured in `_config.yml`)

### Styling Architecture
- SCSS files in `_sass/` are compiled and compressed
- Main entry point: `assets/css/main.scss`
- Bootstrap components can be imported individually from `_sass/bootstrap/`
- Custom components in `_sass/components/`
- Layout-specific styles in `_sass/layout/`

### Homepage Configuration
- Tiles count controlled by `tiles-count` in `_config.yml` (currently 6)
- Tiles can display either pages or posts based on `tiles-source` setting
- Homepage layout defined in `_layouts/home.html`

### Form Integration
Contact forms use Formspree.io. To update the email recipient, modify the `email` field in `_config.yml`.

## Important Considerations

1. **Language**: Site content is primarily in Traditional Chinese (zh-TW)
2. **Images**: Extensive use of images in `assets/images/` - be mindful of file sizes
3. **Mobile Optimization**: Site includes PWA manifest and mobile app icons
4. **SEO**: Chinese meta descriptions configured in `_config.yml`
5. **Social Media**: Facebook and Instagram links in footer (configured in `_config.yml`)