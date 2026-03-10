# BEARY Presentation Tool

**Streamline PowerPoint creation and formatting with AI assistance.**

---

## 🎯 What This Tool Does

The Presentation Tool helps you create professional PowerPoint presentations by:
- Analyzing existing templates to match your style
- Automating slide creation and formatting
- Applying consistent themes and layouts
- Formatting charts, tables, and visual elements

**Important:** This tool handles formatting and structure—your content, message, and storytelling remain yours.

---

## 🚀 Quick Start

### Prerequisites
- Windsurf IDE with Cascade
- Python 3.11+
- PowerPoint templates (optional)

### Installation

1. **Navigate to the presentation tool:**
   ```bash
   cd tools/presentations/powerpoint-mcp-server
   ```

2. **Install dependencies:**
   ```bash
   pip install python-pptx 'mcp[cli]>=1.8.0' Pillow 'fonttools>=4.60.2'
   ```

3. **Configure MCP server in Windsurf:**
   Add this to your Windsurf MCP settings:
   ```json
   {
     "mcpServers": {
       "ppt": {
         "command": "/path/to/python3",
         "args": [
           "/path/to/BEARY/tools/presentations/powerpoint-mcp-server/ppt_mcp_server.py"
         ],
         "env": {
           "PYTHONPATH": "/path/to/BEARY/tools/presentations/powerpoint-mcp-server",
           "PPT_TEMPLATE_PATH": "/path/to/BEARY/tools/presentations/powerpoint-mcp-server/templates"
         }
       }
     }
   }
   ```

4. **Restart Windsurf** to load the MCP server.

---

## 📖 How to Use

### Basic Workflow

Once configured, you can use natural language commands in Windsurf:

```
"Create a new PowerPoint presentation"
"Open the template at /path/to/template.pptx"
"Add a title slide with text 'Q1 Results'"
"Apply professional blue theme"
"Add a bullet list to slide 2"
```

### Example Use Case

```
1. "Open my company template at ~/templates/corporate.pptx"
2. "Extract the style and color scheme"
3. "Create a new presentation based on this template"
4. "Add a title slide: 'Project Update - March 2026'"
5. "Add a content slide with 3 bullet points about our progress"
6. "Save as 'march-update.pptx'"
```

---

## 🛠️ Available Features

### Presentation Management
- Create new presentations
- Open existing files
- Save presentations
- Work with templates

### Content Creation
- Add/modify slides
- Insert text, images, tables
- Create charts and graphs
- Format bullet points

### Professional Design
- Apply themes and color schemes
- Match existing template styles
- Add visual effects
- Format shapes and connectors

### Template Operations
- Analyze template styles
- Extract design elements
- Apply consistent formatting
- Use built-in slide templates

---

## 📁 Directory Structure

```
presentations/
└── powerpoint-mcp-server/
    ├── ppt_mcp_server.py          # Main MCP server
    ├── tools/                      # 32 PowerPoint tools
    ├── utils/                      # Utility functions
    ├── templates/                  # Your PowerPoint templates
    ├── mcp-config.json            # MCP configuration
    └── INSTALLATION_NOTES.md      # Detailed setup guide
```

---

## 🎨 Working with Templates

### Using Your Own Templates

1. **Place templates in the templates folder:**
   ```bash
   cp ~/my-template.pptx tools/presentations/powerpoint-mcp-server/templates/
   ```

2. **Reference in commands:**
   ```
   "Open template 'my-template.pptx'"
   "Create presentation from template 'my-template.pptx'"
   ```

### Extracting Template Styles

```
"Analyze the template and show me the color schemes"
"What layouts are available in this template?"
"Extract the font styles from this presentation"
```

---

## 🔒 Security & Privacy

### ✅ Safe for Internal Documents
- **All processing is local** - Files never leave your machine
- **No cloud uploads** - Everything runs on your computer
- **No network requests** - Fully offline capable

### Security Audit Completed
See `INSTALLATION_NOTES.md` for full security audit details.

---

## ⚙️ Configuration

### Environment Variables

- `PPT_TEMPLATE_PATH` - Directory containing your templates
- `PYTHONPATH` - Path to the MCP server code

### MCP Server Options

The server supports multiple transport modes:
- **stdio** (default) - Standard input/output
- **http** - HTTP server mode
- **sse** - Server-sent events mode

---

## ⚠️ Important Notes

### Repository Status
The PowerPoint MCP server repository is **archived** (as of March 2026):
- ✅ Current version is stable and functional
- ❌ No future updates from original maintainer
- ⚠️ Manual dependency management required

### Maintenance
Check for dependency updates quarterly:
```bash
pip list --outdated
pip install --upgrade python-pptx Pillow fonttools mcp
```

---

## 🔧 Troubleshooting

### MCP Server won't start
1. Check Python path in MCP config
2. Verify all dependencies installed
3. Check `INSTALLATION_NOTES.md` for details

### Template not found
1. Verify `PPT_TEMPLATE_PATH` in config
2. Use absolute paths for templates
3. Check file permissions

### Formatting issues
1. Ensure template is a valid .pptx file
2. Check template has the layouts you need
3. Try with a different template

---

## 📚 Related Documentation

- **[Main BEARY README](../../README.md)** - Overview of all tools
- **[Installation Notes](powerpoint-mcp-server/INSTALLATION_NOTES.md)** - Detailed setup
- **[Security Audit](powerpoint-mcp-server/INSTALLATION_NOTES.md#security-status)** - Security details

---

## 💡 Best Practices

1. **Start with a template** - Easier than building from scratch
2. **Test formatting first** - Verify styles before bulk creation
3. **Use consistent naming** - Makes slides easier to reference
4. **Save frequently** - Prevent loss of work
5. **Keep templates organized** - Use descriptive filenames

---

## 🎯 Use Cases

### When to Use This Tool
- ✅ You have content ready and need to format it
- ✅ You need consistent styling across many slides
- ✅ You're working with an existing template
- ✅ You want to automate repetitive formatting

### When NOT to Use This Tool
- ❌ You need help deciding what to present
- ❌ You want AI to generate presentation content
- ❌ You need complex custom animations
- ❌ You're creating a one-off simple presentation

---

**Remember:** This tool handles the formatting—your content, message, and presentation skills are what make it effective.
