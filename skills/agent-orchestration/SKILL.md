---
name: agent-orchestration
description: Node.js automation scripts, AST-based code generation, and CLI tools for scaffolding Clean Architecture boilerplate from Prisma schemas.
---

# Node.js Automation & AST Expert

Tooling and Developer Experience (DevEx) expert. Builds local Node.js / Bash scripts that automate the repetitive, mechanical parts of Clean Architecture scaffolding.

## When to Activate

- Generating Domain Entities, Repository Interfaces, or Use Case shells from a Prisma schema
- Programmatically wiring a new Use Case into the DI container
- Building a CLI tool to scaffold a complete feature layer (domain → application → infrastructure → presentation)
- Automating repetitive file creation that follows a consistent pattern
- Using `ts-morph` to read, transform, or update existing TypeScript source files
- Syncing Prisma models with the Domain layer after a schema change

## Core Principles

1. **Automate Boilerplate** — Scripts read the Prisma schema (via `@prisma/internals` or `ts-morph`) and scaffold DDD layers automatically.
2. **Code as Data** — Use AST tools (`ts-morph`) to read and modify TypeScript files precisely, rather than fragile string manipulation.
3. **Idiomatic Output** — Generated files must adhere to the Clean Architecture rules defined in the `ts-ddd-clean-architecture` skill.

## Project Setup

```bash
# Place scripts in a dedicated directory
mkdir scripts

# Install tooling as dev dependencies
npm install -D ts-morph @prisma/internals zx handlebars
```

## Script Structure Convention

```
scripts/
├── scaffold-feature.ts      # Main entry: reads Prisma model → generates all layers
├── generators/
│   ├── entity.generator.ts
│   ├── repository-port.generator.ts
│   ├── use-case.generator.ts
│   └── controller.generator.ts
└── templates/
    ├── entity.hbs
    ├── repository-port.hbs
    └── use-case.hbs
```

Add a script to `package.json`:
```json
{
  "scripts": {
    "scaffold": "npx ts-node scripts/scaffold-feature.ts"
  }
}
```

## Reading a Prisma Schema with @prisma/internals

```typescript
// scripts/read-prisma-schema.ts
import { getDMMF } from '@prisma/internals';
import { readFileSync } from 'fs';

async function getPrismaModels() {
  const schema = readFileSync('prisma/schema.prisma', 'utf-8');
  const dmmf = await getDMMF({ datamodel: schema });

  return dmmf.datamodel.models.map((model) => ({
    name: model.name,                            // e.g. "User"
    fields: model.fields.map((f) => ({
      name: f.name,                              // e.g. "email"
      type: f.type,                              // e.g. "String"
      isRequired: f.isRequired,
      isId: f.isId,
    })),
  }));
}

export { getPrismaModels };
```

## Generating Files from Templates (Handlebars)

```typescript
// scripts/generators/entity.generator.ts
import Handlebars from 'handlebars';
import { writeFileSync, mkdirSync } from 'fs';
import { join } from 'path';

const ENTITY_TEMPLATE = `
import { DomainEvent } from '../events/domain-event.interface';

export class {{PascalName}} {
  private readonly domainEvents: DomainEvent[] = [];

  private constructor(
    private readonly id: {{PascalName}}Id,
    {{#each fields}}
    private {{camelName}}: {{tsType}},
    {{/each}}
  ) {}

  static create(id: {{PascalName}}Id, {{constructorParams}}): {{PascalName}} {
    const entity = new {{PascalName}}(id, {{constructorArgs}});
    // TODO: push domain event if needed
    return entity;
  }

  pullDomainEvents(): DomainEvent[] {
    const events = [...this.domainEvents];
    this.domainEvents.length = 0;
    return events;
  }

  getId(): {{PascalName}}Id { return this.id; }
}
`.trim();

interface EntityTemplateData {
  PascalName: string;
  fields: Array<{ camelName: string; tsType: string }>;
  constructorParams: string;
  constructorArgs: string;
}

export function generateEntity(data: EntityTemplateData, outputDir: string): void {
  const template = Handlebars.compile(ENTITY_TEMPLATE);
  const content = template(data);
  const filePath = join(outputDir, `${data.PascalName.toLowerCase()}.entity.ts`);

  mkdirSync(outputDir, { recursive: true });
  writeFileSync(filePath, content, 'utf-8');
  console.log(`✔ Generated: ${filePath}`);
}
```

## Updating Existing Files with ts-morph (AST)

```typescript
// scripts/wire-use-case.ts
// Automatically adds a new Use Case binding to src/main/container.ts
import { Project } from 'ts-morph';

interface WireOptions {
  useCaseName: string;            // e.g. "DeleteUserUseCase"
  repoVariable: string;           // e.g. "userRepo"
  publisherVariable: string;      // e.g. "publisher"
}

export function wireUseCase({ useCaseName, repoVariable, publisherVariable }: WireOptions): void {
  const project = new Project({ tsConfigFilePath: 'tsconfig.json' });
  const containerFile = project.getSourceFileOrThrow('src/main/container.ts');

  // 1. Add the import if it doesn't exist
  const importPath = `../application/use-cases/${toKebabCase(useCaseName)}.use-case`;
  const hasImport = containerFile.getImportDeclarations()
    .some((d) => d.getModuleSpecifierValue() === importPath);

  if (!hasImport) {
    containerFile.addImportDeclaration({
      namedImports: [useCaseName],
      moduleSpecifier: importPath,
    });
  }

  // 2. Append the instantiation at the bottom of the file
  const camelName = useCaseName.charAt(0).toLowerCase() + useCaseName.slice(1);
  containerFile.addStatements(
    `export const ${camelName} = new ${useCaseName}(${repoVariable}, ${publisherVariable});`,
  );

  containerFile.saveSync();
  console.log(`✔ Wired ${useCaseName} into container.ts`);
}

function toKebabCase(str: string): string {
  return str.replace(/([a-z])([A-Z])/g, '$1-$2').toLowerCase();
}
```

## Full Scaffold Script (Orchestrator)

```typescript
// scripts/scaffold-feature.ts
import { getPrismaModels } from './read-prisma-schema';
import { generateEntity } from './generators/entity.generator';
import { generateRepositoryPort } from './generators/repository-port.generator';
import { generateUseCase } from './generators/use-case.generator';

const [,, modelName] = process.argv;
if (!modelName) {
  console.error('Usage: npm run scaffold <ModelName>');
  process.exit(1);
}

async function main() {
  const models = await getPrismaModels();
  const model = models.find((m) => m.name === modelName);

  if (!model) {
    console.error(`Model "${modelName}" not found in schema.prisma`);
    process.exit(1);
  }

  const fields = model.fields
    .filter((f) => !f.isId)
    .map((f) => ({ camelName: f.name, tsType: prismaTypeToTs(f.type) }));

  console.log(`\nScaffolding feature for: ${modelName}`);

  generateEntity({ PascalName: modelName, fields, constructorParams: '', constructorArgs: '' },
    `src/domain/entities`);
  generateRepositoryPort({ PascalName: modelName }, `src/application/ports`);
  generateUseCase({ PascalName: modelName }, `src/application/use-cases`);

  console.log('\n✅ Done. Review generated files and fill in the business logic.');
}

function prismaTypeToTs(prismaType: string): string {
  const map: Record<string, string> = {
    String: 'string', Int: 'number', Float: 'number',
    Boolean: 'boolean', DateTime: 'Date', Json: 'Record<string, unknown>',
  };
  return map[prismaType] ?? prismaType;
}

main().catch(console.error);
```

## Execution Workflow

When asked to build an automation script:
1. Identify the **input source** (Prisma model name, JSON config, or manual args)
2. Write the **parsing logic** (use `@prisma/internals` for Prisma schema)
3. Write the **generator functions** (Handlebars templates for file content)
4. Write the **AST wiring step** (`ts-morph` to update `container.ts`)
5. Register the script in `package.json` under `"scripts"`

## Anti-Patterns

| ❌ Never Do | ✅ Instead |
|---|---|
| String-replace existing TypeScript files | Use `ts-morph` AST transformations |
| Hardcode file paths inside generator functions | Pass `outputDir` as a parameter |
| Generate files that violate Clean Architecture layers | Follow the workflow in `ts-ddd-clean-architecture` skill |
| Run generators in `src/` directly | Keep scripts in `scripts/` directory; never in `src/` |
| Commit generated boilerplate blindly | Review and fill in business logic before committing |
