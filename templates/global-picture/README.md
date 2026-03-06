# global-picture/ — Template Structure

This folder shows the directory structure that Milo's `/define-standards` workflow will create at the **project root** (not inside the `milo/` folder).

When `/define-standards` runs, it creates:

```
{project-root}/global-picture/
├── standards/
│   ├── coding-standards.md       ← generated from milo/templates/standards-doc.md
│   ├── oss-packages.md           ← generated from milo/templates/standards-doc.md
│   └── architectural-patterns.md ← generated from milo/templates/standards-doc.md
├── api-docs/
│   └── README.md                 ← auto-created; module docs added by /module-summary
├── features/
│   ├── registry.md               ← feature tracking table
│   └── README.md
└── oss-registry/
    └── registry.json             ← initialized from milo/knowledge/oss-registry-schema.md
```

This `templates/global-picture/` folder is a reference only — it is not copied directly. The workflow generates the actual files with real content based on the human's answers.
