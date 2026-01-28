# Contributing to MCP Tool Shop Website

Thank you for your interest in contributing to the MCP Tool Shop website! This is the organization's GitHub Pages site showcasing our tools and documentation.

## Development Setup

```bash
git clone https://github.com/mcp-tool-shop/mcp-tool-shop.github.io.git
cd mcp-tool-shop.github.io
```

The site is static HTML/CSS/JavaScript, so you can simply open the HTML files in a browser or use a local server:

```bash
# Using Python
python -m http.server 8000

# Using Node.js
npx http-server

# Using PHP
php -S localhost:8000
```

Then visit `http://localhost:8000`

## How to Contribute

### Reporting Issues

If you find a bug or have a suggestion:

1. Check existing [issues](https://github.com/mcp-tool-shop/mcp-tool-shop.github.io/issues)
2. If not found, create a new issue with:
   - Clear description of the problem or suggestion
   - Browser and OS information (for bugs)
   - Screenshots if relevant
   - Steps to reproduce (for bugs)

### Contributing Code

1. **Fork the repository** and create a branch from `main`
2. **Make your changes**
   - Follow existing HTML/CSS/JS conventions
   - Ensure responsive design works on mobile
   - Test in multiple browsers
3. **Test your changes locally**
   - Check all links work
   - Verify responsive layouts
   - Test on different screen sizes
4. **Commit your changes**
   - Use clear, descriptive commit messages
   - Reference issue numbers when applicable
5. **Submit a pull request**
   - Describe what your PR does and why
   - Include screenshots for UI changes
   - Link to related issues

## Project Structure

```
mcp-tool-shop.github.io/
├── index.html              # Homepage
├── images/                 # Image assets
├── performance/            # Performance analytics
├── comfy-headless.html     # Tool pages
├── dev-brain.html
├── file-compass.html
├── tool-compass.html
├── voice-soundboard.html
├── context-bar.html
├── context-window-manager.html
├── claude-fresh.html
├── CNAME                   # Custom domain config
└── README-perf.md          # Performance documentation
```

## Adding a New Tool Page

When adding a new tool to the site:

1. **Create a new HTML page** (e.g., `my-tool.html`)
2. **Use consistent structure**:
   - Hero section with tool name and description
   - Features section
   - Installation instructions
   - Usage examples
   - Links to GitHub repository
3. **Add to navigation** in `index.html`
4. **Include meta tags** for SEO
5. **Add screenshots** in `images/`
6. **Test responsive layout**

## Design Guidelines

### Visual Design
- Maintain consistent color scheme
- Use readable font sizes (minimum 16px body text)
- Ensure sufficient color contrast (WCAG AA minimum)
- Keep layouts clean and uncluttered

### Responsive Design
- Mobile-first approach
- Test on common breakpoints: 320px, 768px, 1024px, 1440px
- Use flexible layouts (flexbox, grid)
- Optimize images for different screen sizes

### Performance
- Minimize file sizes
- Optimize images (WebP when possible)
- Minimize HTTP requests
- Use lazy loading for images
- Inline critical CSS

### Accessibility
- Use semantic HTML
- Include alt text for all images
- Ensure keyboard navigation works
- Use ARIA labels where appropriate
- Test with screen readers

## Testing

Before submitting a PR, test:

### Cross-Browser Testing
- Chrome/Edge (Chromium)
- Firefox
- Safari (macOS/iOS)

### Responsive Testing
- Mobile devices (iOS, Android)
- Tablets
- Desktop (various screen sizes)

### Accessibility Testing
- Keyboard navigation
- Screen reader compatibility (NVDA, JAWS, VoiceOver)
- Color contrast validation
- HTML validation (W3C validator)

## Performance Monitoring

The site includes performance analytics in the `performance/` directory. When making changes:

1. Check baseline performance
2. Make your changes
3. Compare new performance metrics
4. Document any significant changes

## Content Guidelines

### Writing Style
- Clear, concise language
- Technical but approachable
- Active voice preferred
- Avoid jargon where possible

### Documentation
- Include code examples
- Provide installation steps
- Show common use cases
- Link to GitHub repositories

### Links
- Keep internal links relative
- Test all external links
- Use descriptive link text (avoid "click here")

## Deployment

The site deploys automatically via GitHub Pages when changes are pushed to the `main` branch. Changes typically appear within a few minutes.

### Custom Domain
The site uses a custom domain configured via `CNAME`. Don't modify this file unless changing the domain.

## Common Tasks

### Updating Tool Information

Edit the relevant HTML file and update:
- Version numbers
- Feature lists
- Installation instructions
- Screenshots

### Adding Images

1. Optimize images before adding (use WebP or optimized PNG/JPG)
2. Place in `images/` directory
3. Use descriptive filenames
4. Include alt text in HTML

### Updating Styles

- Keep styles consistent with existing design
- Use CSS custom properties (variables) when available
- Consider mobile layouts
- Test in multiple browsers

## Questions?

Open an issue or start a discussion in the [MCP Tool Shop](https://github.com/mcp-tool-shop) organization.
