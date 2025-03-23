# Detailed Implementation Plan: GetPage Web & Clipboard Capture Command

## 1. Overview

This document outlines the implementation plan for extending the GetPage CLI tool with a new command to capture and store web content and clipboard data as markdown files.

## 2. Command Structure

I propose adding a new command called `capture` with the following structure:

```
getPage capture [OPTIONS] [SOURCE]
```

### Options:
- `--url URL`: URL to capture (mutually exclusive with `--clipboard`)
- `--clipboard`: Capture clipboard content (mutually exclusive with `--url`)
- `--output FILE_PATH`: Full path to the output file (cannot be used with `--folder`)
- `--folder FOLDER_PATH`: Folder path to save the markdown file (cannot be used with `--output`)
- `--profile NAME`: LLM profile to use (optional, will use default if not specified)
- `--scope`, `--local`, `--global`, `--file`: Standard scope options for configuration
- `--format`: Output format option (table or JSON)

## 3. Implementation Components

### 3.1 Profile Management

Create a specialized profile manager for content capture:

```python
# getPage/extensibility/capture_extension.py

from getPage.utils.profiles import BaseProfileManager, ProfileValidationResult
from getPage.extensibility.generator import ProfileCommandGenerator

# Define capture profile parameters
CAPTURE_PROFILE_PARAMS = [
    {"name": "name", "type": str, "help": "Profile name", "required": True},
    {"name": "format_template", "type": str, "help": "Markdown template format", "required": False},
    {"name": "filename_template", "type": str, "help": "Template for generated filenames", "required": False},
    {"name": "cleanup_level", "type": str, "help": "Content cleanup level (minimal, moderate, thorough)", "required": False},
    {"name": "extract_metadata", "type": bool, "help": "Whether to extract metadata from content", "required": False},
]

class CaptureProfileManager(BaseProfileManager):
    """Specialized profile manager for content capture profiles."""

    def __init__(self):
        """Initialize a capture profile manager."""
        super().__init__("capture", CAPTURE_PROFILE_PARAMS)
    
    def _validate_field_values(self, profile: Dict[str, Any]) -> ProfileValidationResult:
        """Validate capture-specific field values."""
        errors = []
        
        # Validate cleanup_level if specified
        if "cleanup_level" in profile:
            level = profile["cleanup_level"]
            valid_levels = ["minimal", "moderate", "thorough"]
            if level not in valid_levels:
                errors.append(f"Cleanup level must be one of: {', '.join(valid_levels)}")
        
        return {
            "valid": len(errors) == 0,
            "errors": errors,
            "profile": profile
        }
    
    def _apply_default_values(self, profile: Dict[str, Any]) -> Dict[str, Any]:
        """Apply default values for capture profiles."""
        defaults = {
            "format_template": "# {title}\n\n{content}\n\n---\nSource: {source}\nCaptured: {date}",
            "filename_template": "{title}",
            "cleanup_level": "moderate",
            "extract_metadata": True
        }
        
        for field, default_value in defaults.items():
            if field not in profile:
                profile[field] = default_value
                
        return profile

# Factory function for the profile manager
def get_capture_profile_manager() -> CaptureProfileManager:
    """Factory function to create a capture profile manager."""
    return CaptureProfileManager()

# Command generator for capture profiles
capture_profile_command_generator = ProfileCommandGenerator(
    name="Capture",
    command_name="capture-profile",
    description="Manage content capture profiles for formatting and processing web/clipboard content.",
    profile_params=CAPTURE_PROFILE_PARAMS,
    profile_manager_factory=get_capture_profile_manager,
    help_texts={
        "create": "Create a new content capture profile with formatting templates and processing options.",
        "list": "List all available content capture profiles.",
        "show": "Show details for a specific content capture profile.",
        "edit": "Edit an existing content capture profile.",
        "delete": "Delete a content capture profile.",
        "use": "Set a specific content capture profile as the default."
    }
)

# Generate the command group for profile management
capture_profile_group = capture_profile_command_generator.generate_command_group()
```

### 3.2 Config Integration

Update the default configuration in `config.py` to include capture profiles:

```python
DEFAULT_CONFIG = {
    "profiles": {
        "llm": {},
        "database": {},
        "capture": {}  # Add capture profiles
    },
    "defaults": {
        "llm": None,
        "database": None,
        "capture": None  # Add default capture profile
    },
    "settings": {
        "output_format": "json",
        "color_theme": "dark",
        "log_level": "info",
        "capture_dir": "~/Documents/captures"  # Default directory for captures
    }
}
```

### 3.3 Capture Command Implementation

Create a new module for the capture command:

```python
# getPage/commands/capture_cmd.py

import os
import sys
import json
import datetime
import click
import pyperclip
import requests
from typing import Optional, Dict, Any

from getPage.utils.context import _initialize_context
from getPage.utils.formatting import OutputFormatter
from getPage.utils.profiles import ProfileManager
from getPage.commands.cmd_options import scope_options, json_format_option
from getPage.utils.command_registry import CommandRegistry

# LLM integration for content processing
class CaptureContentProcessor:
    """Processes captured content using LLM capabilities."""
    
    def __init__(self, llm_profile=None, capture_profile=None):
        """Initialize with specified profiles."""
        # Get profile managers
        self.llm_manager = ProfileManager("llm")
        self.capture_manager = ProfileManager("capture")
        
        # Get profiles
        self.llm_profile = self._get_llm_profile(llm_profile)
        self.capture_profile = self._get_capture_profile(capture_profile)
    
    def _get_llm_profile(self, profile_name=None):
        """Get the LLM profile to use."""
        if profile_name:
            return self.llm_manager.get_profile(profile_name)
        
        # Try to get default profile
        default_name = self.llm_manager.get_default_profile()
        if default_name:
            return self.llm_manager.get_profile(default_name)
        
        # No profile found
        raise ValueError("No LLM profile specified and no default profile found")
    
    def _get_capture_profile(self, profile_name=None):
        """Get the capture profile to use."""
        if profile_name:
            return self.capture_manager.get_profile(profile_name)
        
        # Try to get default profile
        default_name = self.capture_manager.get_default_profile()
        if default_name:
            return self.capture_manager.get_profile(default_name)
        
        # Use built-in defaults if no profile found
        return {
            "format_template": "# {title}\n\n{content}\n\n---\nSource: {source}\nCaptured: {date}",
            "filename_template": "{title}",
            "cleanup_level": "moderate",
            "extract_metadata": True
        }
    
    def process_web_content(self, url: str) -> Dict[str, Any]:
        """Process web content using LLM."""
        # This would integrate with your LLM using the configured profile
        # For now, we'll outline the structure

        # 1. Use LLM to extract and format the content
        # 2. Apply the template from the capture profile
        # 3. Return formatted content and metadata

        # Placeholder implementation
        return {
            "title": "Extracted title",
            "content": "Processed markdown content...",
            "source": url,
            "date": datetime.datetime.now().isoformat(),
            "metadata": {}
        }
    
    def process_clipboard_content(self) -> Dict[str, Any]:
        """Process clipboard content using LLM."""
        # Similar to web content processing but for clipboard text
        clipboard_text = pyperclip.paste()
        
        if not clipboard_text:
            raise ValueError("Clipboard is empty")
        
        # Process with LLM similar to web content

        # Placeholder implementation
        return {
            "title": "Clipboard content",
            "content": clipboard_text,
            "source": "clipboard",
            "date": datetime.datetime.now().isoformat(),
            "metadata": {}
        }
    
    def generate_filename(self, data: Dict[str, Any]) -> str:
        """Generate a filename based on the content and template."""
        # Use the filename template from the capture profile
        template = self.capture_profile.get("filename_template", "{title}")
        
        # Basic string formatting with the data
        filename = template.format(**data)
        
        # Clean up the filename to make it safe for filesystem
        filename = self._sanitize_filename(filename)
        
        # Ensure it has .md extension
        if not filename.endswith(".md"):
            filename += ".md"
        
        return filename
    
    def _sanitize_filename(self, filename: str) -> str:
        """Make a string safe to use as a filename."""
        # Replace problematic characters
        for char in ['/', '\\', ':', '*', '?', '"', '<', '>', '|']:
            filename = filename.replace(char, '_')
        
        # Limit length
        if len(filename) > 100:
            filename = filename[:97] + "..."
        
        return filename
    
    def format_markdown(self, data: Dict[str, Any]) -> str:
        """Format the data into markdown using the template."""
        template = self.capture_profile.get("format_template", 
                                          "# {title}\n\n{content}\n\n---\nSource: {source}\nCaptured: {date}")
        
        return template.format(**data)


@click.command(name="capture")
@click.option("--url", help="URL to capture content from")
@click.option("--clipboard", is_flag=True, help="Capture content from clipboard")
@click.option("--output", help="Output file path (including filename)")
@click.option("--folder", help="Output folder path (filename will be generated)")
@click.option("--profile", help="LLM profile to use for processing")
@click.option("--capture-profile", help="Capture profile to use for formatting")
@scope_options
@json_format_option
def capture_cmd(url, clipboard, output, folder, profile, capture_profile, 
                scope, file_path, output_format):
    """Capture web or clipboard content and store as markdown."""
    
    # Initialize context
    scope_params = {
        "scope": scope,
        "file_path": file_path
    }
    ctx = _initialize_context(scope_params)
    
    try:
        # Validate input combinations
        if url and clipboard:
            raise ValueError("Cannot specify both URL and clipboard. Choose one source.")
        
        if not url and not clipboard:
            raise ValueError("Must specify either URL or clipboard as the content source.")
        
        if output and folder:
            raise ValueError("Cannot specify both output file and folder. Choose one destination.")
        
        # Set up content processor
        processor = CaptureContentProcessor(profile, capture_profile)
        
        # Process content based on source
        if url:
            content_data = processor.process_web_content(url)
        else:  # clipboard
            content_data = processor.process_clipboard_content()
        
        # Format markdown content
        markdown_content = processor.format_markdown(content_data)
        
        # Determine output path
        output_path = None
        if output:
            # Use specified output path
            output_path = os.path.expanduser(output)
        elif folder:
            # Generate filename and combine with folder
            folder_path = os.path.expanduser(folder)
            filename = processor.generate_filename(content_data)
            output_path = os.path.join(folder_path, filename)
        else:
            # Use default location from settings
            settings = ctx.settings
            default_dir = settings.get_setting("capture_dir") or "~/Documents/captures"
            folder_path = os.path.expanduser(default_dir)
            
            # Ensure directory exists
            os.makedirs(folder_path, exist_ok=True)
            
            # Generate filename
            filename = processor.generate_filename(content_data)
            output_path = os.path.join(folder_path, filename)
        
        # Ensure output directory exists
        os.makedirs(os.path.dirname(output_path), exist_ok=True)
        
        # Write content to file
        with open(output_path, 'w', encoding='utf-8') as f:
            f.write(markdown_content)
        
        # Output success message
        if output_format == "json":
            result = {
                "status": "success",
                "output_file": output_path,
                "source": "url" if url else "clipboard",
                "characters": len(markdown_content),
                "timestamp": datetime.datetime.now().isoformat()
            }
            OutputFormatter.print_json(result, "Capture Result")
        else:
            OutputFormatter.print_success(f"Content captured successfully to: {output_path}")
    
    except (ValueError, FileNotFoundError, requests.RequestException) as e:
        OutputFormatter.print_error(str(e))
```

### 3.4 Main Integration

Update `main.py` to register the new commands:

```python
# Add imports
from getPage.commands.capture_cmd import capture_cmd
from getPage.extensibility.capture_extension import capture_profile_group

# Add command groups
cli.add_command(capture_cmd)
cli.add_command(capture_profile_group)
```

## 4. LLM Integration

For LLM integration, the implementation will use the existing LLM profile management that's already in place. The key functions that will use LLM include:

1. **Content Extraction & Formatting**: When processing web content, the LLM will be used to:
   - Extract the main content from HTML
   - Format it as clean markdown
   - Identify the title and other metadata

2. **Clipboard Processing**: When processing clipboard content, the LLM will:
   - Determine content type (plain text, HTML, code, etc.)
   - Format appropriately as markdown
   - Generate a title based on the content

3. **Filename Generation**: If no output file is specified, the LLM will:
   - Generate an appropriate filename based on content
   - Ensure the filename is filesystem-safe

## 5. Package Dependencies

Add the following dependencies to `pyproject.toml`:

```toml
dependencies = [
    # ... existing dependencies
    "requests>=2.28.0",
    "pyperclip>=1.8.2",
    "beautifulsoup4>=4.11.0",
]
```

## 6. Implementation Steps

1. **Setup Capture Profiles**:
   - Implement the capture profile manager
   - Add capture profile command group
   - Extend configuration system to include capture profiles

2. **LLM Content Processing**:
   - Create helper classes for content processing
   - Implement web content extraction and formatting
   - Implement clipboard content processing

3. **Command Implementation**:
   - Implement the capture command with all options
   - Add file handling and path resolution
   - Integrate with LLM and capture profiles

4. **Testing & Documentation**:
   - Create test cases for various capture scenarios
   - Document the command and its options
   - Update README with usage examples

## 7. Usage Examples

```bash
# Capture a web page and save to default location with auto-generated filename
getPage capture --url https://example.com

# Capture clipboard content to a specific file
getPage capture --clipboard --output ~/Documents/notes/my-capture.md

# Capture web page with specific LLM profile and save to a specific folder
getPage capture --url https://example.com --profile "gpt4-standard" --folder ~/Documents/webpages

# Use a specific capture profile for formatting
getPage capture --url https://example.com --capture-profile "academic-format"

# Create a new capture profile
getPage capture-profile create --name "blog-format" --format-template "# {title}\n\n{content}\n\n---\nSource: {source}" --cleanup-level "thorough"
```

## 8. Conclusion

This implementation plan provides a detailed framework for extending the GetPage CLI tool with web and clipboard capture functionality. The design follows the existing patterns and standards of the GetPage tool, leveraging its configuration and profile management capabilities while adding new functionality in a modular, maintainable way.

The implementation makes extensive use of LLM capabilities for intelligent content processing while giving users fine-grained control through profile settings and command options.