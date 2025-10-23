# Google Drive PDF Screenshot Exporter

A JavaScript console script to export Google Drive documents as optimized, high-quality PDFs with working hyperlinks when the download option is disabled.

## ⚠️ Disclaimer

**This script is intended for personal use only.** Only use it on documents you own or have explicit permission to download.

## What It Does

When you open a Google Drive document (PDF, presentation, etc.) that has downloads disabled, this script:
1. Captures all visible page images at high resolution
2. Extracts clickable hyperlinks from the document
3. Generates an optimized PDF with custom page sizes (no white margins)
4. Adds working hyperlinks to the PDF
5. Compresses the output for reasonable file sizes
6. Downloads the resulting PDF

## Prerequisites

- Google Chrome browser (recommended) or any Chromium-based browser
- The document must be viewable in Google Drive (you need viewing permissions)

## How to Use

### Step 1: Open the Document
Navigate to the Google Drive document you want to export and ensure it's fully loaded in the preview. **Scroll through the entire document** to ensure all pages are loaded.

### Step 2: Open Developer Console
- **Windows/Linux**: Press `F12` or `Ctrl + Shift + J`
- **Mac**: Press `Cmd + Option + J`

### Step 3: Paste and Run the Script
Copy the entire script below, paste it into the console, and press `Enter`.

```javascript
(async function() {
    console.log("=== Ultimate PDF Extractor ===");
    console.log("Features: High quality • Working links • Optimized compression\n");
    
    // Configuration
    const CONFIG = {
        jpegQuality: 0.92,      // High quality but compressed (0.92 is sweet spot)
        dpi: 72,                // Standard screen DPI for good file size
        pdfCompression: true,   // Enable PDF compression
        progressIndicator: true // Show progress on screen
    };
    
    // Load jsPDF
    const loadScript = (src) => {
        return new Promise((resolve, reject) => {
            let script = document.createElement('script');
            script.onload = resolve;
            script.onerror = reject;
            
            if (window.trustedTypes && trustedTypes.createPolicy) {
                const policy = trustedTypes.createPolicy('ultimatePolicy_' + Date.now(), {
                    createScriptURL: (input) => input
                });
                script.src = policy.createScriptURL(src);
            } else {
                script.src = src;
            }
            
            document.body.appendChild(script);
        });
    };
    
    console.log("Loading jsPDF library...");
    await loadScript('https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js');
    const { jsPDF } = window.jspdf;
    console.log("✓ jsPDF loaded\n");
    
    // Create progress indicator
    let progressDiv = null;
    if (CONFIG.progressIndicator) {
        progressDiv = document.createElement('div');
        progressDiv.style.cssText = `
            position: fixed;
            top: 20px;
            right: 20px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            padding: 20px 25px;
            border-radius: 12px;
            box-shadow: 0 10px 40px rgba(0,0,0,0.3);
            z-index: 10000;
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
            min-width: 320px;
        `;
        
        // Create elements programmatically to avoid TrustedHTML issues
        let title = document.createElement('div');
        title.style.cssText = 'font-size: 18px; font-weight: 600; margin-bottom: 12px;';
        title.textContent = '📄 Creating PDF';
        
        let progressText = document.createElement('div');
        progressText.id = 'progress-text';
        progressText.style.cssText = 'font-size: 14px; margin-bottom: 8px; opacity: 0.95;';
        progressText.textContent = 'Initializing...';
        
        let progressBarContainer = document.createElement('div');
        progressBarContainer.style.cssText = 'background: rgba(255,255,255,0.2); height: 6px; border-radius: 3px; overflow: hidden; margin-bottom: 8px;';
        
        let progressBar = document.createElement('div');
        progressBar.id = 'progress-bar';
        progressBar.style.cssText = 'height: 100%; background: white; width: 0%; transition: width 0.3s ease;';
        progressBarContainer.appendChild(progressBar);
        
        let progressStats = document.createElement('div');
        progressStats.id = 'progress-stats';
        progressStats.style.cssText = 'font-size: 12px; opacity: 0.85;';
        progressStats.textContent = 'Pages: 0 • Links: 0';
        
        progressDiv.appendChild(title);
        progressDiv.appendChild(progressText);
        progressDiv.appendChild(progressBarContainer);
        progressDiv.appendChild(progressStats);
        
        document.body.appendChild(progressDiv);
    }
    
    const updateProgress = (text, percent, pages, links) => {
        if (!progressDiv) return;
        document.getElementById('progress-text').innerText = text;
        document.getElementById('progress-bar').style.width = percent + '%';
        if (pages !== undefined) {
            document.getElementById('progress-stats').innerText = `Pages: ${pages} • Links: ${links}`;
        }
    };
    
    // Extract all links from document
    updateProgress('Finding links...', 5);
    console.log("=== Extracting Links ===");
    
    let allLinks = Array.from(document.querySelectorAll('a[href]'));
    console.log(`Found ${allLinks.length} links in document`);
    
    let linkData = allLinks.map(link => {
        let rect = link.getBoundingClientRect();
        
        // Extract text from link (multiple fallbacks)
        let text = link.innerText || link.textContent || '';
        
        if (!text.trim()) {
            let children = link.querySelectorAll('span, div, p, h1, h2, h3, h4, h5, h6');
            if (children.length > 0) {
                text = Array.from(children)
                    .map(el => el.innerText || el.textContent)
                    .filter(t => t && t.trim())
                    .join(' ');
            }
        }
        
        if (!text.trim()) {
            text = link.getAttribute('aria-label') || 
                   link.getAttribute('title') || 
                   `[${link.href.split('/').pop() || 'Link'}]`;
        }
        
        return {
            href: link.href,
            text: text.trim().substring(0, 100),
            rect: {
                left: rect.left,
                top: rect.top,
                right: rect.right,
                bottom: rect.bottom,
                width: rect.width,
                height: rect.height
            }
        };
    }).filter(link => link.rect.width > 0 && link.rect.height > 0);
    
    console.log(`Processed ${linkData.length} valid links\n`);
    
    // Get all page images
    updateProgress('Finding pages...', 10);
    let images = Array.from(document.querySelectorAll('img[src^="blob:"]'));
    
    if (images.length === 0) {
        alert("❌ No images found! Make sure the document is fully loaded.");
        if (progressDiv) progressDiv.remove();
        return;
    }
    
    console.log(`Found ${images.length} pages\n`);
    
    let pdf = null;
    let totalLinksAdded = 0;
    let startTime = Date.now();
    
    // Process each page
    for (let pageIndex = 0; pageIndex < images.length; pageIndex++) {
        let img = images[pageIndex];
        let percentComplete = ((pageIndex / images.length) * 85 + 10).toFixed(0); // 10-95%
        
        updateProgress(
            `Processing page ${pageIndex + 1} of ${images.length}...`,
            percentComplete,
            pageIndex + 1,
            totalLinksAdded
        );
        
        console.log(`Page ${pageIndex + 1}/${images.length}`);
        
        // Get image dimensions
        let imgWidth = img.naturalWidth;
        let imgHeight = img.naturalHeight;
        let imgRect = img.getBoundingClientRect();
        
        // Create high-quality canvas
        let canvas = document.createElement('canvas');
        let ctx = canvas.getContext('2d', { alpha: false });
        canvas.width = imgWidth;
        canvas.height = imgHeight;
        
        // High-quality rendering settings
        ctx.imageSmoothingEnabled = true;
        ctx.imageSmoothingQuality = 'high';
        ctx.drawImage(img, 0, 0, imgWidth, imgHeight);
        
        // Get optimized image data
        let imgData = canvas.toDataURL('image/jpeg', CONFIG.jpegQuality);
        
        // Calculate PDF dimensions
        let widthInMM = (imgWidth * 25.4) / CONFIG.dpi;
        let heightInMM = (imgHeight * 25.4) / CONFIG.dpi;
        let orientation = imgWidth > imgHeight ? 'l' : 'p';
        
        // Find links overlapping this page
        let pageLinks = linkData.filter(link => {
            return !(link.rect.right < imgRect.left || 
                    link.rect.left > imgRect.right ||
                    link.rect.bottom < imgRect.top ||
                    link.rect.top > imgRect.bottom);
        });
        
        // Create or add page
        if (pageIndex === 0) {
            pdf = new jsPDF({
                orientation: orientation,
                unit: 'mm',
                format: [widthInMM, heightInMM],
                compress: CONFIG.pdfCompression
            });
        } else {
            pdf.addPage([widthInMM, heightInMM], orientation);
        }
        
        // Add image to page
        pdf.addImage(imgData, 'JPEG', 0, 0, widthInMM, heightInMM);
        
        // Add clickable links
        for (let link of pageLinks) {
            try {
                // Calculate relative position within image
                let relX = Math.max(0, link.rect.left - imgRect.left);
                let relY = Math.max(0, link.rect.top - imgRect.top);
                let relWidth = Math.min(link.rect.width, imgRect.right - link.rect.left);
                let relHeight = Math.min(link.rect.height, imgRect.bottom - link.rect.top);
                
                // Convert to PDF coordinates
                let pdfX = (relX / imgRect.width) * widthInMM;
                let pdfY = (relY / imgRect.height) * heightInMM;
                let pdfWidth = (relWidth / imgRect.width) * widthInMM;
                let pdfHeight = (relHeight / imgRect.height) * heightInMM;
                
                // Add link annotation
                pdf.link(pdfX, pdfY, pdfWidth, pdfHeight, { url: link.href });
                
                totalLinksAdded++;
                console.log(`  ✓ Link: ${link.text.substring(0, 40)}...`);
                
            } catch (e) {
                console.log(`  ✗ Failed to add link: ${e.message}`);
            }
        }
    }
    
    // Finalize
    let elapsedTime = ((Date.now() - startTime) / 1000).toFixed(1);
    
    updateProgress('Saving PDF...', 95, images.length, totalLinksAdded);
    console.log(`\n=== Saving PDF ===`);
    
    // Save the PDF
    pdf.save('document.pdf');
    
    // Completion
    updateProgress('Complete! ✓', 100, images.length, totalLinksAdded);
    
    setTimeout(() => {
        if (progressDiv) progressDiv.remove();
    }, 3000);
    
    // Summary
    console.log(`\n=== Complete ===`);
    console.log(`✓ Pages processed: ${images.length}`);
    console.log(`✓ Links added: ${totalLinksAdded}`);
    console.log(`✓ Time taken: ${elapsedTime}s`);
    console.log(`✓ JPEG quality: ${(CONFIG.jpegQuality * 100).toFixed(0)}%`);
    console.log(`✓ PDF compression: ${CONFIG.pdfCompression ? 'Enabled' : 'Disabled'}`);
    
    alert(`PDF Created Successfully!\n\n` +
          `File saved as: document.pdf`);
    
})();
```

### Step 4: Download
The PDF will automatically download as `document.pdf` to your default downloads folder.

## Tips for Better Quality

1. **Scroll through the entire document first** to ensure all pages are loaded
2. **Zoom in** before running the script - Drive loads higher resolution images at higher zoom levels (try 150% or 200%)
3. **Wait for loading** - Make sure all page thumbnails have finished loading
4. **Check console logs** - The script outputs resolution info for each page

## Troubleshooting

### Script doesn't run
- Check for any console messages asking to type a specific command to allow pasting code into the console
- Make sure you copied the entire script
- Check if the browser console shows any error messages
- Try refreshing the page and running again

### Output only contains the first few pages of the original document
- Scroll through all pages first to load the images. Ensure they are fully loaded and visible before running the script
- Try refreshing the page and waiting for it to fully load

### "No images found!" alert
- Make sure you're on a Google Drive document preview page (not the file list)
- Scroll through all pages first to load the images. Ensure they are fully loaded and visible before running the script
- Try refreshing the page and waiting for it to fully load

### Low quality output
- Zoom in to 150-200% before running the script
- Scroll through all pages to force Drive to load higher resolution images
- The quality is limited by what Google Drive serves in the browser

## Technical Details

- Uses jsPDF 2.5.1 for PDF generation with compression
- Captures images at native resolution (naturalWidth/Height)
- JPEG quality set to 0.92 (optimized for quality vs file size)
- Extracts and preserves clickable hyperlinks
- Custom page sizing prevents white margins
- 72 DPI conversion for optimal print quality

## Limitations

- Only captures what's visible in the browser (limited by Drive's image serving)
- Text is rasterized (not selectable in the PDF)
- Quality depends on Google Drive's image resolution

## License

MIT License - Use at your own risk and responsibility.
