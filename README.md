# Stacked Systems

## An Engineer's AI Journey

This repository contains the Jekyll-based blog "Stacked Systems" focused on homelab infrastructure, systems engineering, and AI-augmented workflows.

## Blog Overview

This blog documents personal tech experiments, infrastructure projects, systems engineering builds, and AI-augmented workflows using tools like:

- Proxmox VE
- TrueNAS Scale with ZFS
- Kubernetes
- Model Context Protocol (MCP)
- Claude AI for content management

## Technical Details

- Built with Jekyll and GitHub Pages
- Managed through MCP using Claude AI
- Responsive design for all devices
- Category and tag-based organization

## Setup Instructions

To set up this blog locally or deploy it to GitHub Pages:

### Local Development

1. Clone this repository
   ```bash
   git clone https://github.com/eddygk/stacked-systems.git
   cd stacked-systems
   ```

2. Install dependencies
   ```bash
   gem install bundler jekyll
   bundle install
   ```

3. Start the local Jekyll server
   ```bash
   bundle exec jekyll serve
   ```

4. Visit `http://localhost:4000/stacked-systems/` in your browser

### GitHub Pages Deployment

1. Go to the repository settings
2. Navigate to the "Pages" section in the left sidebar
3. Under "Source", select "Deploy from a branch"
4. Choose "main" branch and "/" (root) directory
5. Click "Save"
6. Wait a few minutes for the site to be built and deployed
7. Visit `https://eddygk.github.io/stacked-systems/` to see the live site

## Adding New Content

### Creating a New Post

1. Create a new file in the `_posts` directory following the naming convention `YYYY-MM-DD-title.md`
2. Add the following front matter at the top of the file:
   ```yaml
   ---
   layout: post
   title: "Your Post Title"
   date: YYYY-MM-DD
   categories: [category1, category2]
   tags: [tag1, tag2, tag3]
   ---
   ```
3. Write your post content in Markdown format
4. Commit and push to the main branch

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Acknowledgments

- Built with [Jekyll](https://jekyllrb.com/)
- Managed with [Claude AI](https://claude.ai)
- Hosted on [GitHub Pages](https://pages.github.com/)

---

Created and maintained using Claude AI and Model Context Protocol (MCP)
