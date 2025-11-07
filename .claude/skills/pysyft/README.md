# PySyft Claude Skill

This skill provides expert assistance for working with PySyft, a privacy-preserving data science platform.

## Installation

### For Claude Code

The skill is automatically available when you open this repository in Claude Code.

### For Claude Desktop

1. Copy the `pysyft` directory to your Claude skills folder:
   ```bash
   # macOS
   cp -r .claude/skills/pysyft ~/Library/Application\ Support/Claude/skills/

   # Linux
   cp -r .claude/skills/pysyft ~/.config/Claude/skills/

   # Windows
   cp -r .claude/skills/pysyft %APPDATA%/Claude/skills/
   ```

2. Restart Claude Desktop

## Usage

### In Claude Code

Simply ask questions about PySyft and Claude will automatically use this skill:

```
"How do I create a datasite in PySyft?"
"Show me how to upload a dataset with twin objects"
"Help me write a syft_function with differential privacy"
```

### In Claude Desktop

Invoke the skill by mentioning PySyft or privacy-preserving ML:

```
"I'm working with PySyft, can you help me set up a server?"
"How do I implement a custom output policy in PySyft?"
```

## What This Skill Provides

- **Quick Reference**: Installation, server setup, client connection
- **Code Patterns**: Common workflows for data scientists and data owners
- **Best Practices**: Privacy-preserving code, error handling, testing
- **Troubleshooting**: Common issues and solutions
- **Examples**: Healthcare, finance, research use cases
- **API Reference**: All major functions and their usage

## Skill Capabilities

This skill helps with:

✓ Setting up PySyft servers (datasite, gateway, enclave)
✓ Creating and managing datasets with twin objects
✓ Writing syft_functions for remote execution
✓ Implementing privacy policies (input and output)
✓ Deploying PySyft (development, Docker, Kubernetes)
✓ Debugging PySyft applications
✓ Integrating differential privacy (OpenDP)
✓ Multi-party computation workflows
✓ Request and approval workflows

## Examples

### Setting Up a Development Server

Ask: *"How do I set up a PySyft development server?"*

Claude will provide:
```python
import syft as sy

server = sy.orchestra.launch(
    name="dev-server",
    port=8080,
    server_type="datasite",
    dev_mode=True,
    reset=True
)

client = sy.login(
    url="localhost",
    port=8080,
    email="info@openmined.org",
    password="changethis"
)
```

### Uploading Data

Ask: *"Show me how to upload a dataset with mock and private data"*

Claude will provide complete code with explanations.

### Writing Privacy-Preserving Functions

Ask: *"How do I write a syft_function that uses differential privacy?"*

Claude will provide DP-enabled code using OpenDP.

## Customization

You can customize this skill by editing `skill.md`:

1. Add your own common patterns
2. Include organization-specific examples
3. Add custom policies or configurations
4. Include internal deployment guidelines

## Documentation References

- **PySyft Docs**: https://docs.openmined.org
- **GitHub**: https://github.com/OpenMined/PySyft
- **API Reference**: https://docs.openmined.org/api
- **Tutorials**: See `/notebooks` in the PySyft repository

## Troubleshooting

If the skill isn't working:

1. **Claude Code**: Ensure you're in the PySyft repository directory
2. **Claude Desktop**: Verify the skill is in the correct skills directory
3. **Permissions**: Ensure the skill files are readable
4. **Restart**: Restart Claude Code or Claude Desktop

## Contributing

To improve this skill:

1. Edit `skill.md` with new patterns or examples
2. Add common questions to the FAQ section
3. Update version information when PySyft updates
4. Submit improvements via pull request

## Version

**Skill Version**: 1.0.0
**PySyft Version**: 0.9.6-beta.6
**Last Updated**: 2025-11-07

## License

This skill documentation is part of PySyft and follows the same Apache 2.0 license.
