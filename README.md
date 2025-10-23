# Google Drive PDF Screenshot Exporter

A JavaScript console script to export Google Drive documents as high-quality PDFs when the download option is disabled.

## ⚠️ Disclaimer

**This script is intended for personal use only.** Only use it on documents you own or have explicit permission to download.

## What It Does

When you open a Google Drive document (PDF, presentation, etc.) that has downloads disabled, this script:
1. Captures all visible page images
2. Converts them to high-resolution format
3. Generates a PDF with custom page sizes (no white margins)
4. Downloads the resulting PDF

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
let trustedURL;
if (window.trustedTypes && trustedTypes.createPolicy) {
    const policy = trustedTypes.createPolicy('myPolicy', {
        createScriptURL: (input) => input
    });
    trustedURL = policy.createScriptURL('https://cdnjs.cloudflare.com/ajax/libs/jspdf/1.3.2/jspdf.min.js');
} else {
    trustedURL = 'https://cdnjs.cloudflare.com/ajax/libs/jspdf/1.3.2/jspdf.min.js';
}

let jspdf = document.createElement("script");
jspdf.onload = function () {
    let elements = document.getElementsByTagName("img");
    let validImages = [];
    
    // Filter and collect valid blob images
    for (let i = 0; i < elements.length; i++) {
        if (/^blob:/.test(elements[i].src)) {
            validImages.push(elements[i]);
        }
    }
    
    if (validImages.length === 0) {
        alert("No images found!");
        return;
    }
    
    console.log(`Found ${validImages.length} images to process`);
    let pdf = null;
    
    validImages.forEach((img, index) => {
        // Use naturalWidth/Height for actual image dimensions
        let imgWidth = img.naturalWidth;
        let imgHeight = img.naturalHeight;
        
        console.log(`Image ${index + 1}: ${imgWidth}x${imgHeight}px`);
        
        // Create high-resolution canvas
        let canvasElement = document.createElement('canvas');
        let con = canvasElement.getContext("2d", { alpha: false });
        
        // Set canvas to actual image dimensions
        canvasElement.width = imgWidth;
        canvasElement.height = imgHeight;
        
        // Improve rendering quality
        con.imageSmoothingEnabled = true;
        con.imageSmoothingQuality = 'high';
        
        // Draw image at full resolution
        con.drawImage(img, 0, 0, imgWidth, imgHeight);
        
        // Get image data at maximum quality (1.0 = lossless)
        let imgData = canvasElement.toDataURL("image/jpeg", 1.0);
        
        // Convert pixels to mm (using 72 DPI for better quality)
        let widthInMM = (imgWidth * 25.4) / 72;
        let heightInMM = (imgHeight * 25.4) / 72;
        
        // Determine orientation
        let orientation = imgWidth > imgHeight ? 'l' : 'p';
        
        if (index === 0) {
            // Create first page with custom dimensions
            pdf = new jsPDF(orientation, 'mm', [widthInMM, heightInMM]);
        } else {
            // Add new page with custom dimensions for this image
            pdf.addPage([widthInMM, heightInMM], orientation);
        }
        
        // Add image to fill the entire page (no margins)
        pdf.addImage(imgData, 'JPEG', 0, 0, widthInMM, heightInMM);
    });
    
    pdf.save("download.pdf");
    alert("PDF generated successfully with " + validImages.length + " pages!");
};

jspdf.src = trustedURL;
document.body.appendChild(jspdf);
```

### Step 4: Download
The PDF will automatically download as `download.pdf` to your default downloads folder.

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

- Uses jsPDF 1.3.2 for PDF generation
- Captures images at native resolution (naturalWidth/Height)
- JPEG quality set to 1.0 (maximum)
- Custom page sizing prevents white margins
- 72 DPI conversion for optimal print quality

## Limitations

- Only captures what's visible in the browser (limited by Drive's image serving)
- Text is rasterized (not selectable in the PDF)
- Quality depends on Google Drive's image resolution

## License

MIT License - Use at your own risk and responsibility.
