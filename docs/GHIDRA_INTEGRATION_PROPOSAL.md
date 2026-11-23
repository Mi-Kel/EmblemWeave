# Ghidra Integration Proposal for EmblemWeave

## Executive Summary

**Status**: ✅ **DISCOVERED** - Ghidra 12.1 installed at `X:\REA\ghidra`
**Date**: 2026-11-20
**Integration Priority**: HIGH - Reverse engineering is core to EmblemWeave's lift/recompile pipeline

---

## Discovered Environment

| Component | Version | Path | Notes |
|-----------|---------|------|-------|
| Ghidra | 12.1 (PUBLIC) | `X:\REA\ghidra` | Build date: 2026-05-13 |
| JDK 17 | (bundled) | `X:\REA\jdk17` | Required for Ghidra |
| JDK 21 | (bundled) | `X:\REA\jdk21` | Alternative runtime |
| Gradle | (bundled) | `X:\REA\.gradle` | For building extensions |
| Python | 3.11-3.14 supported | `X:\REA\ghidra\Ghidra\application.properties` | Ghidra supports Python 3.9-3.14 |

---

## Integration Architecture

### 1. Ghidra Headless Analyzer (Primary Integration Point)

The **Ghidra Headless Analyzer** (`analyzeHeadless`) is the command-line interface for automated analysis:

```bash
# Windows batch
X:\REA\ghidra\ghidraRun.bat analyzeHeadless <project_dir> <project_name> -import <binary> -scriptPath <scripts> -postScript <script>

# Or via Ghidra installation
X:\REA\ghidra\Ghidra\support\analyzeHeadless.bat ...
```

**Capabilities**:
- Import binaries (NSO, ELF, PE, Mach-O)
- Run analysis passes (disassembly, decompilation, type propagation)
- Execute Java/Python scripts for custom analysis
- Export results (JSON, XML, C headers, Ghidra project)

### 2. Python API (ghidra_bridge / pyhidra)

For tighter integration with EmblemWeave's Python pipeline:

```python
# Using ghidra_bridge (recommended)
import ghidra_bridge
bridge = ghidra_bridge.GhidraBridge(namespace=globals())
flat = bridge.getFlatAPI()
# Use Ghidra API directly from Python

# Or pyhidra (embedded)
import pyhidra
pyhidra.start()
from ghidra.app.decompiler import DecompInterface
```

### 3. Java Extensions (Ghidra Extensions)

Custom analyzers/loaders can be built as Ghidra extensions:
- Location: `X:\REA\ghidra\Extensions\`
- Build with Gradle: `./gradlew buildExtension`

---

## Proposed Integration Points

### A. NSO Binary Analysis Pipeline

```
Game NSO → Ghidra Headless → Function/Type Recovery → Lift to C → EmblemWeave Recompile
```

**Implementation**:
1. Create Ghidra project for each game module
2. Run headless analysis with Nintendo Switch loader
3. Extract function signatures, type definitions, control flow
4. Export to intermediate format (JSON/Protobuf) for lifter

### B. Decompilation-Assisted Lifting

```python
# Python script for Ghidra headless
from ghidra.app.decompiler import DecompInterface
from ghidra.util.task import ConsoleTaskMonitor

def decompile_function(func):
    decompiler = DecompInterface()
    decompiler.openProgram(currentProgram)
    result = decompiler.decompileFunction(func, 60, ConsoleTaskMonitor())
    if result.decompileCompleted():
        return result.getDecompiledFunction().getC()
    return None
```

**Use Cases**:
- Recover high-level C from ARM64 assembly
- Identify struct layouts and vtables
- Extract string references and constants

### C. Type Library Generation

Ghidra can generate `.ghidra` type archives or C headers:
- Feed into EmblemWeave's type system
- Validate lifted code against recovered types
- Enable cross-referencing between games

---

## Implementation Plan

### Phase 1: Infrastructure (Week 1)
- [ ] Add `ghidra` tool to `tools.lock` with path `X:\REA\ghidra`
- [ ] Create `GhidraManager` in `toolchain_manager.rs`
- [ ] Add `analyzeHeadless` wrapper in Python/Rust

### Phase 2: NSO Analysis (Week 2)
- [ ] Implement Nintendo Switch ARM64 loader configuration
- [ ] Create analysis scripts for function recovery
- [ ] Export function signatures to JSON

### Phase 3: Lifter Integration (Week 3)
- [ ] Connect Ghidra decompilation output to `nxrecomp` lifter
- [ ] Add type propagation from Ghidra to lifted C
- [ ] Validate round-trip: NSO → Ghidra → C → Recompile → NSO

### Phase 4: Automation (Week 4)
- [ ] Integrate into `engine.rs` build pipeline
- [ ] Add structured events for Ghidra stages
- [ ] Cache analysis results via sccache-compatible mechanism

---

## Toolchain Manifest Entry

```json
{
  "name": "ghidra",
  "display_name": "Ghidra Reverse Engineering Suite",
  "category": "optional",
  "version": "12.1",
  "min_version": "11.0",
  "platform": "all",
  "source": {
    "type": "system",
    "hint": "Install from X:\\REA\\ghidra or https://ghidra-sre.org"
  },
  "expected_path": "X:\\REA\\ghidra",
  "sha256": null,
  "archive_sha256": null,
  "version_cmd": ["ghidraRun.bat", "--version"],
  "version_regex": "Ghidra (\\d+\\.\\d+\\.\\d+) ",
  "on_path": false,
  "env_vars": {
    "GHIDRA_INSTALL_DIR": "X:\\REA\\ghidra",
    "JAVA_HOME": "X:\\REA\\jdk21"
  },
  "depends_on": [],
  "description": "NSA reverse engineering framework for NSO analysis and decompilation"
}
```

---

## Security Considerations

- Ghidra runs Java - ensure JDK is updated
- Headless mode has no GUI attack surface
- Validate all binary inputs before analysis
- Run analysis in isolated/sandboxed environment if possible

---

## Resources

- [Ghidra Documentation](https://ghidra-sre.org/Documentation.html)
- [Ghidra API Javadoc](https://ghidra-sre.org/api/)
- [ghidra_bridge Python Package](https://github.com/justfoxing/ghidra_bridge)
- [pyhidra](https://github.com/boringcat/pyhidra)
- [Nintendo Switch Loader](https://github.com/Repo-Friendly/ghidra-nso-loader) (community)

---

## Sign-Off

**Discovery**: Complete - Ghidra 12.1 confirmed at `X:\REA\ghidra`
**Proposal**: Ready for Phase C implementation
**Priority**: High - Core to lift/recompile pipeline

*Generated: 2026-11-20 | Phase B Validation*