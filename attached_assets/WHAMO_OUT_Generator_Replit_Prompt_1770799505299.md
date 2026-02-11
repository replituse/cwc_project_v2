# Replit Project Enhancement: WHAMO .OUT File Generator with File Upload

## Project Overview
This is a hydraulic network analysis application that needs to integrate the WHAMO (Water Hammer and Mass Oscillation) engine to process user-uploaded .INP files and generate corresponding .OUT analysis files.

## Feature Request: "Generate .OUT" Button with File Upload

### 1. User Flow
1. User clicks the "Generate .OUT" button
2. System opens a file manager/file picker dialog
3. User selects a `.inp` file from their computer
4. System uploads the selected .inp file
5. Backend processes the file using WHAMO.EXE engine
6. System generates a .out file with hydraulic analysis results
7. System automatically downloads the generated .out file to user's computer
8. (Optional) Display success notification with file name

### 2. UI Requirements

#### Button Specifications
- **Label:** "Generate .OUT"
- **Location:** [Specify location in your UI - e.g., top toolbar, sidebar, etc.]
- **Icon (optional):** Upload/process icon (⬆️ or ⚙️)
- **Style:** Match existing application design system
- **State Management:**
  - Default state: Enabled and clickable
  - Processing state: Show loading spinner with "Processing..." text
  - Disabled state: If processing is in progress

#### File Picker Dialog
- **Accept:** Only `.inp` files (filter by extension)
- **File size limit:** Maximum 10MB (adjust as needed)
- **Single file selection:** Only one .inp file at a time

### 3. Backend Implementation Requirements

#### Technology Stack Integration
**WHAMO.EXE Engine:**
- Executable location: Store in `/server/engines/` or similar protected directory
- Engine type: Windows executable (may require Wine/compatibility layer on Linux servers)
- Version: 3.1 (as per sample files)

#### API Endpoint Specification

**Endpoint:** `POST /api/generate-out`

**Request:**
- Method: POST
- Content-Type: multipart/form-data
- Body: 
  - `inpFile`: File (the uploaded .inp file)
  - (Optional) `projectName`: String

**Response (Success - 200):**
```json
{
  "success": true,
  "message": "OUT file generated successfully",
  "fileName": "project_output.out",
  "downloadUrl": "/api/download/[fileId]",
  "fileSize": 61440
}
```

**Response (Error - 400/500):**
```json
{
  "success": false,
  "error": "Error message",
  "details": "Detailed error information"
}
```

#### Processing Pipeline

```
1. FILE UPLOAD
   ↓
   - Receive .inp file from client
   - Validate file extension (.inp only)
   - Validate file size (< 10MB)
   - Generate unique temporary file ID
   - Save to temp directory: /tmp/whamo_[uniqueId]/input.inp
   
2. WHAMO.EXE EXECUTION
   ↓
   - Navigate to temp directory
   - Execute: WHAMO.EXE input.inp
   - Wait for process completion
   - Capture stdout/stderr for debugging
   - Monitor for timeout (max 60 seconds)
   
3. OUTPUT CAPTURE
   ↓
   - Locate generated output.out file
   - Validate output file exists and is not empty
   - Read output file content
   - Rename to: [originalName]_output.out
   
4. FILE DELIVERY
   ↓
   - Store in download directory (or stream directly)
   - Generate download URL/token
   - Return response to client
   - Trigger automatic download
   
5. CLEANUP
   ↓
   - Delete temporary files after 1 hour
   - Clean up old temp directories
   - Log completion
```

### 4. WHAMO.EXE Integration Details

#### Command Line Usage
```bash
# Basic execution (WHAMO typically reads from stdin or expects input.inp in same directory)
WHAMO.EXE < input.inp > output.out

# OR (if WHAMO accepts file argument)
WHAMO.EXE input.inp

# OR (if WHAMO looks for specific filename)
# Rename uploaded file to expected name, execute WHAMO, then read output
```

#### Expected Input File Format (.INP)
Based on the sample `1.inp` file:

```
c Project Name
C SYSTEM CONNECTIVITY

SYSTEM
ELEM HW AT 1
ELEM C1 LINK 1 2
JUNCTION AT 2
ELEM D1 LINK 2 100
[... more elements ...]

NODE 1 ELEV 4022.31
[... more nodes ...]

FINISH

C ELEMENT PROPERTIES
RESERVOIR
 ID HW
 ELEV 4130.58
FINISH

CONDUIT ID C1 LENG 13405.51 DIAM 34.45 CELE 2852.51 FRIC 0.008
[... more conduits ...]

SURGETANK
 ID ST SIMPLE
 ELTOP 4215.88
[... more properties ...]

FLOWBC ID FB1 QSCHEDULE 1 FINISH
[... more boundary conditions ...]

SCHEDULE
 QSCHEDULE 1 T 0 Q 3000 T 20 Q 0 T 3000 Q 0
FINISH

C OUTPUT REQUEST
HISTORY
 NODE 2 Q HEAD
 ELEM ST Q ELEV
FINISH

C COMPUTATIONAL PARAMETERS
CONTROL
 DTCOMP 0.01 DTOUT .1 TMAX 500.0
FINISH

C EXECUTION CONTROL
GO
GOODBYE
```

#### Expected Output File Format (.OUT)
Based on the sample `1_OUT.OUT` file:

- Text file with analysis results
- Contains header with WHAMO version info
- Echo of input file
- Computational results (pressure, flow, head time histories)
- Approximately 1100+ lines for typical analysis
- File size: ~60KB for the sample

### 5. Frontend Implementation

#### JavaScript/TypeScript Example

```javascript
// Button click handler
async function handleGenerateOut() {
  // Create file input element
  const fileInput = document.createElement('input');
  fileInput.type = 'file';
  fileInput.accept = '.inp';
  
  // Handle file selection
  fileInput.onchange = async (e) => {
    const file = e.target.files[0];
    
    if (!file) return;
    
    // Validate file extension
    if (!file.name.endsWith('.inp')) {
      alert('Please select a valid .inp file');
      return;
    }
    
    // Show loading state
    setLoading(true);
    
    try {
      // Create form data
      const formData = new FormData();
      formData.append('inpFile', file);
      
      // Call API
      const response = await fetch('/api/generate-out', {
        method: 'POST',
        body: formData
      });
      
      if (!response.ok) {
        throw new Error('Failed to generate OUT file');
      }
      
      // Get the blob
      const blob = await response.blob();
      
      // Trigger download
      const url = window.URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = file.name.replace('.inp', '_output.out');
      document.body.appendChild(a);
      a.click();
      document.body.removeChild(a);
      window.URL.revokeObjectURL(url);
      
      // Show success message
      showNotification('OUT file generated successfully!', 'success');
      
    } catch (error) {
      console.error('Error:', error);
      showNotification('Failed to generate OUT file. Please try again.', 'error');
    } finally {
      setLoading(false);
    }
  };
  
  // Trigger file picker
  fileInput.click();
}
```

#### React Example

```jsx
import React, { useState } from 'react';

function GenerateOutButton() {
  const [isLoading, setIsLoading] = useState(false);
  
  const handleFileSelect = async (event) => {
    const file = event.target.files[0];
    if (!file) return;
    
    setIsLoading(true);
    
    const formData = new FormData();
    formData.append('inpFile', file);
    
    try {
      const response = await fetch('/api/generate-out', {
        method: 'POST',
        body: formData
      });
      
      if (!response.ok) throw new Error('Generation failed');
      
      const blob = await response.blob();
      const url = window.URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = file.name.replace('.inp', '_output.out');
      a.click();
      window.URL.revokeObjectURL(url);
      
    } catch (error) {
      alert('Error generating OUT file: ' + error.message);
    } finally {
      setIsLoading(false);
    }
  };
  
  return (
    <div>
      <input
        type="file"
        accept=".inp"
        onChange={handleFileSelect}
        style={{ display: 'none' }}
        id="inp-file-input"
      />
      <label htmlFor="inp-file-input">
        <button 
          as="span"
          disabled={isLoading}
          onClick={() => document.getElementById('inp-file-input').click()}
        >
          {isLoading ? 'Processing...' : 'Generate .OUT'}
        </button>
      </label>
    </div>
  );
}
```

### 6. Backend Implementation (Node.js Example)

```javascript
const express = require('express');
const multer = require('multer');
const { exec } = require('child_process');
const path = require('path');
const fs = require('fs');
const { v4: uuidv4 } = require('uuid');

const app = express();
const upload = multer({ dest: 'uploads/' });

app.post('/api/generate-out', upload.single('inpFile'), async (req, res) => {
  const tempId = uuidv4();
  const tempDir = path.join(__dirname, 'temp', tempId);
  
  try {
    // Create temp directory
    fs.mkdirSync(tempDir, { recursive: true });
    
    // Copy uploaded file to temp directory
    const inputPath = path.join(tempDir, 'input.inp');
    fs.copyFileSync(req.file.path, inputPath);
    
    // Path to WHAMO.EXE
    const whamoPath = path.join(__dirname, 'engines', 'WHAMO.EXE');
    
    // Execute WHAMO.EXE
    // Note: On Linux, you may need Wine: wine WHAMO.EXE
    const command = process.platform === 'win32' 
      ? `cd ${tempDir} && ${whamoPath} < input.inp > output.out`
      : `cd ${tempDir} && wine ${whamoPath} < input.inp > output.out`;
    
    exec(command, { timeout: 60000 }, (error, stdout, stderr) => {
      if (error) {
        console.error('WHAMO execution error:', error);
        return res.status(500).json({ 
          success: false, 
          error: 'Failed to process file',
          details: error.message 
        });
      }
      
      // Read output file
      const outputPath = path.join(tempDir, 'output.out');
      
      if (!fs.existsSync(outputPath)) {
        return res.status(500).json({ 
          success: false, 
          error: 'Output file not generated' 
        });
      }
      
      // Send file to client
      const originalName = req.file.originalname.replace('.inp', '');
      res.download(outputPath, `${originalName}_output.out`, (err) => {
        // Cleanup
        fs.unlinkSync(req.file.path);
        fs.rmSync(tempDir, { recursive: true, force: true });
        
        if (err) {
          console.error('Download error:', err);
        }
      });
    });
    
  } catch (error) {
    console.error('Error:', error);
    res.status(500).json({ 
      success: false, 
      error: 'Internal server error',
      details: error.message 
    });
  }
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

### 7. Error Handling & Validation

#### Client-Side Validation
- ✅ Check file extension is `.inp`
- ✅ Validate file size (max 10MB)
- ✅ Ensure file is not empty
- ✅ Show user-friendly error messages

#### Server-Side Validation
- ✅ Verify file upload successful
- ✅ Validate .inp file format (basic syntax check)
- ✅ Check WHAMO.EXE exists and is executable
- ✅ Handle WHAMO.EXE execution errors
- ✅ Timeout long-running processes (>60s)
- ✅ Verify output file was created

#### Common Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| "Invalid file format" | Wrong file type uploaded | Accept only .inp files |
| "WHAMO.EXE not found" | Engine not in correct path | Verify WHAMO.EXE location |
| "Permission denied" | WHAMO.EXE not executable | chmod +x WHAMO.EXE |
| "Output file empty" | Invalid .inp syntax | Validate .inp before processing |
| "Process timeout" | Very large network | Increase timeout limit |
| "Wine not installed" | Linux server missing Wine | Install Wine for Windows exe support |

### 8. Platform Compatibility

#### Windows Server
- ✅ WHAMO.EXE runs natively
- ✅ No additional dependencies needed

#### Linux Server (Replit)
- ⚠️ Requires Wine to run Windows executables
- Install Wine: `apt-get install wine` or `apt-get install wine64`
- Execute with: `wine WHAMO.EXE < input.inp > output.out`
- May have performance overhead

#### Alternative: Docker Container
```dockerfile
FROM ubuntu:20.04
RUN apt-get update && apt-get install -y wine wine64
COPY WHAMO.EXE /app/engines/
WORKDIR /app
```

### 9. File Naming Convention

**Input Files (uploaded by user):**
- User's original filename (e.g., `project1.inp`)

**Output Files (generated):**
- Pattern: `[originalName]_output.out`
- Example: If user uploads `project1.inp`, generate `project1_output.out`
- Alternative with timestamp: `project1_20260211_143052.out`

### 10. Optional Enhancements

#### Progress Indicators
- Show upload progress bar
- Display "Processing..." with spinner during WHAMO execution
- Estimated time remaining (if calculable)

#### File Preview
- Option to preview .out file in browser before download
- Syntax highlighting for .inp and .out files

#### Batch Processing
- Allow multiple .inp files to be uploaded at once
- Generate multiple .out files in parallel
- Download as ZIP archive

#### History/Logging
- Keep history of processed files (optional)
- Allow users to re-download recent .out files
- Log processing time and file sizes

### 11. Security Considerations

⚠️ **Important Security Measures:**

1. **File Upload Security:**
   - Validate file extensions (whitelist only .inp)
   - Limit file size (prevent DoS)
   - Scan uploaded files for malicious content
   - Use unique random filenames for storage

2. **Process Execution Security:**
   - Run WHAMO.EXE in sandboxed environment
   - Set execution timeouts
   - Limit concurrent executions
   - Validate output file before sending to user

3. **File System Security:**
   - Store temp files in isolated directory
   - Clean up temp files regularly
   - Set proper file permissions
   - Prevent directory traversal attacks

### 12. Testing Checklist

- [ ] File picker opens on button click
- [ ] Only .inp files can be selected
- [ ] File upload works correctly
- [ ] WHAMO.EXE executes successfully
- [ ] .out file is generated with correct content
- [ ] .out file downloads automatically
- [ ] Proper filename is used for download
- [ ] Loading states display correctly
- [ ] Error messages are user-friendly
- [ ] Multiple sequential generations work
- [ ] Large files process without timeout
- [ ] Cleanup removes temporary files
- [ ] Works on target deployment platform (Windows/Linux)

## Sample Files Provided

1. **WHAMO.EXE** (2MB) - The hydraulic analysis engine
2. **1.inp** (3.5KB) - Sample input file for testing
3. **1_OUT.OUT** (60KB) - Expected output format

## Acceptance Criteria

✅ User can click "Generate .OUT" button  
✅ File picker dialog opens and accepts only .inp files  
✅ Selected .inp file is uploaded to server  
✅ WHAMO.EXE processes the .inp file successfully  
✅ Generated .out file matches the format of sample `1_OUT.OUT`  
✅ .out file automatically downloads to user's computer  
✅ Appropriate loading/success/error states are shown  
✅ Process completes within 60 seconds for typical files  
✅ Temporary files are cleaned up after processing  
✅ Works reliably on Replit deployment environment  

## Implementation Priority
**Priority:** High  
**Complexity:** Medium  
**Estimated Time:** 4-8 hours (depending on platform compatibility)

## Questions to Address Before Implementation

1. **Platform:** Is your Replit environment Windows or Linux-based?
2. **Existing Stack:** What is your current tech stack (React, Node.js, Python, etc.)?
3. **File Storage:** Where should temporary files be stored?
4. **Concurrent Users:** How many simultaneous users do you expect?
5. **File Size Limits:** What is the maximum expected .inp file size?
6. **Cleanup Strategy:** How long should generated .out files be retained?

---

**Ready to implement?** This prompt provides everything needed to add the WHAMO .OUT file generation feature to your project.
