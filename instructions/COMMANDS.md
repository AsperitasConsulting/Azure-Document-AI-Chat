# Command templates

> Claude-Flow install

```
npx claude-flow init --force
```
Add ```**/.claude-flow/metrics/*``` to ```.gitignore```

> Command prompt base
```
npx claude-flow swarm "" --claude
```

> Plan instruction
```
npx claude-flow swarm "Create plan implementation plan for application described in INFRASTRUCTURE.md in folder instructions. Place the implementation plan in folder plans. Do not implement the feature additions yet. Please let me know if you need additional information." --claude
```

> Plan instruction
```
npx claude-flow swarm "Implement requested features described in INFRASTRUCTURE.md in folder instructions. Please follow the implementation plans you created in folder plans. Please let me know if you need additional information." --claude
```

