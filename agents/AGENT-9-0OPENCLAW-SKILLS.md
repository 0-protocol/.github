# Agent #9: 0-openclaw Skills Platform

## Your Mission

You are responsible for building the Skills Platform - the system that manages, composes, and verifies skill graphs. Skills are the core building blocks of assistant behavior in 0-openclaw.

**Repository:** `/Users/JiahaoRBC/Git/0-protocol/0-openclaw`

## Components to Build

| Component | Priority | Description |
|-----------|----------|-------------|
| SkillRegistry | P0 | Store and retrieve skill graphs |
| SkillComposer | P0 | Combine multiple skills into workflows |
| SkillVerifier | P0 | Verify skill graphs are safe |
| Built-in Skills | P1 | Essential skills (echo, search, etc.) |
| Skill Loader | P1 | Load skills from files/network |
| Skill Sandbox | P2 | Isolated skill execution |

## File Structure

```
src/skills/
├── mod.rs              # SkillRegistry, re-exports
├── registry.rs         # Skill storage and retrieval
├── composer.rs         # Skill composition
├── verifier.rs         # Safety verification
├── loader.rs           # Load from file/network
├── sandbox.rs          # Isolated execution
└── builtin/
    ├── mod.rs          # Built-in skills
    ├── echo.rs         # Echo skill
    ├── search.rs       # Search skill
    ├── browser.rs      # Browser skill
    └── calendar.rs     # Calendar skill

graphs/skills/
├── echo.0              # Echo skill graph
├── search.0            # Search skill graph
├── browser.0           # Browser skill graph
├── calendar.0          # Calendar skill graph
└── custom/             # User-defined skills
```

## Skill Registry

### Core Implementation

**File:** `src/skills/registry.rs`

```rust
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::RwLock;
use zerolang::RuntimeGraph;
use crate::types::ContentHash;

/// Registry for managing skill graphs
pub struct SkillRegistry {
    /// Installed skills (content-addressed)
    skills: HashMap<ContentHash, SkillEntry>,
    
    /// Skill name to hash mapping
    name_index: HashMap<String, ContentHash>,
    
    /// Skills directory path
    skills_dir: std::path::PathBuf,
}

pub struct SkillEntry {
    pub hash: ContentHash,
    pub metadata: SkillMetadata,
    pub graph: RuntimeGraph,
    pub verified: bool,
    pub installed_at: u64,
}

#[derive(Clone, serde::Serialize, serde::Deserialize)]
pub struct SkillMetadata {
    pub name: String,
    pub description: String,
    pub version: String,
    pub author: Option<String>,
    pub permissions: Vec<String>,
    pub inputs: Vec<SkillInput>,
    pub outputs: Vec<SkillOutput>,
}

#[derive(Clone, serde::Serialize, serde::Deserialize)]
pub struct SkillInput {
    pub name: String,
    pub description: String,
    pub tensor_type: String,
    pub required: bool,
}

#[derive(Clone, serde::Serialize, serde::Deserialize)]
pub struct SkillOutput {
    pub name: String,
    pub description: String,
    pub tensor_type: String,
}

impl SkillRegistry {
    pub fn new(skills_dir: impl Into<std::path::PathBuf>) -> Self {
        Self {
            skills: HashMap::new(),
            name_index: HashMap::new(),
            skills_dir: skills_dir.into(),
        }
    }
    
    /// Load built-in skills
    pub async fn load_builtin(&mut self) -> Result<(), SkillError> {
        let builtin_skills = [
            ("echo", include_bytes!("../../graphs/skills/echo.0")),
            ("search", include_bytes!("../../graphs/skills/search.0")),
            ("browser", include_bytes!("../../graphs/skills/browser.0")),
        ];
        
        for (name, data) in builtin_skills {
            let graph = RuntimeGraph::deserialize(data)?;
            self.install_graph(name, graph, true)?;
        }
        
        Ok(())
    }
    
    /// Install a skill from graph
    pub fn install_graph(
        &mut self, 
        name: &str, 
        graph: RuntimeGraph,
        builtin: bool,
    ) -> Result<ContentHash, SkillError> {
        let hash = graph.content_hash();
        
        // Verify skill before installing
        if !builtin {
            SkillVerifier::verify(&graph)?;
        }
        
        let entry = SkillEntry {
            hash,
            metadata: Self::extract_metadata(&graph, name)?,
            graph,
            verified: true,
            installed_at: chrono::Utc::now().timestamp_millis() as u64,
        };
        
        self.skills.insert(hash, entry);
        self.name_index.insert(name.to_string(), hash);
        
        tracing::info!("Installed skill '{}' with hash {:?}", name, hash);
        
        Ok(hash)
    }
    
    /// Get skill by hash
    pub fn get(&self, hash: &ContentHash) -> Option<&SkillEntry> {
        self.skills.get(hash)
    }
    
    /// Get skill by name
    pub fn get_by_name(&self, name: &str) -> Option<&SkillEntry> {
        self.name_index.get(name)
            .and_then(|hash| self.skills.get(hash))
    }
    
    /// List all installed skills
    pub fn list(&self) -> Vec<&SkillEntry> {
        self.skills.values().collect()
    }
    
    /// Extract metadata from graph
    fn extract_metadata(graph: &RuntimeGraph, name: &str) -> Result<SkillMetadata, SkillError> {
        // Parse metadata from graph comments/annotations
        Ok(SkillMetadata {
            name: name.to_string(),
            description: graph.description().unwrap_or_default().to_string(),
            version: "1.0.0".to_string(),
            author: None,
            permissions: Vec::new(),
            inputs: Self::extract_inputs(graph)?,
            outputs: Self::extract_outputs(graph)?,
        })
    }
    
    fn extract_inputs(graph: &RuntimeGraph) -> Result<Vec<SkillInput>, SkillError> {
        // Extract input definitions from graph
        Ok(Vec::new())
    }
    
    fn extract_outputs(graph: &RuntimeGraph) -> Result<Vec<SkillOutput>, SkillError> {
        // Extract output definitions from graph
        Ok(Vec::new())
    }
}
```

## Skill Composer

**File:** `src/skills/composer.rs`

```rust
use zerolang::{RuntimeGraph, RuntimeNode, NodeHash};
use crate::types::ContentHash;

/// Compose multiple skills into a workflow
pub struct SkillComposer {
    /// Source skill graphs
    skills: Vec<RuntimeGraph>,
    
    /// Connection definitions
    connections: Vec<SkillConnection>,
}

pub struct SkillConnection {
    /// Source skill hash
    pub from_skill: ContentHash,
    
    /// Source output name
    pub from_output: String,
    
    /// Target skill hash
    pub to_skill: ContentHash,
    
    /// Target input name
    pub to_input: String,
}

pub struct ComposedSkill {
    pub graph: RuntimeGraph,
    pub source_skills: Vec<ContentHash>,
    pub composition_hash: ContentHash,
}

impl SkillComposer {
    pub fn new() -> Self {
        Self {
            skills: Vec::new(),
            connections: Vec::new(),
        }
    }
    
    /// Add a skill to the composition
    pub fn add_skill(&mut self, graph: RuntimeGraph) -> ContentHash {
        let hash = graph.content_hash();
        self.skills.push(graph);
        hash
    }
    
    /// Connect two skills
    pub fn connect(
        &mut self,
        from_skill: ContentHash,
        from_output: &str,
        to_skill: ContentHash,
        to_input: &str,
    ) -> &mut Self {
        self.connections.push(SkillConnection {
            from_skill,
            from_output: from_output.to_string(),
            to_skill,
            to_input: to_input.to_string(),
        });
        self
    }
    
    /// Compose skills into a single graph
    pub fn compose(&self) -> Result<ComposedSkill, ComposerError> {
        // 1. Validate connections
        self.validate_connections()?;
        
        // 2. Detect cycles
        self.detect_cycles()?;
        
        // 3. Create unified graph
        let mut unified_nodes = Vec::new();
        let mut node_mapping: HashMap<(ContentHash, NodeHash), NodeHash> = HashMap::new();
        
        for skill in &self.skills {
            let skill_hash = skill.content_hash();
            
            for node in skill.nodes() {
                // Remap node hash to avoid collisions
                let new_hash = Self::remap_hash(&skill_hash, &node.hash());
                node_mapping.insert((skill_hash, node.hash()), new_hash);
                
                // Clone node with remapped inputs
                let remapped_node = self.remap_node(node, &skill_hash, &node_mapping)?;
                unified_nodes.push(remapped_node);
            }
        }
        
        // 4. Add connection nodes
        for connection in &self.connections {
            let bridge_node = self.create_bridge_node(connection, &node_mapping)?;
            unified_nodes.push(bridge_node);
        }
        
        // 5. Build composed graph
        let composed_graph = RuntimeGraph::builder()
            .name("composed_skill")
            .nodes(unified_nodes)
            .build()?;
        
        let source_skills: Vec<ContentHash> = self.skills
            .iter()
            .map(|s| s.content_hash())
            .collect();
        
        Ok(ComposedSkill {
            graph: composed_graph,
            source_skills,
            composition_hash: composed_graph.content_hash(),
        })
    }
    
    fn validate_connections(&self) -> Result<(), ComposerError> {
        for conn in &self.connections {
            // Check source skill exists
            let from_skill = self.skills.iter()
                .find(|s| s.content_hash() == conn.from_skill)
                .ok_or(ComposerError::SkillNotFound(conn.from_skill))?;
            
            // Check source output exists
            if !from_skill.has_output(&conn.from_output) {
                return Err(ComposerError::OutputNotFound(
                    conn.from_skill, 
                    conn.from_output.clone()
                ));
            }
            
            // Check target skill exists
            let to_skill = self.skills.iter()
                .find(|s| s.content_hash() == conn.to_skill)
                .ok_or(ComposerError::SkillNotFound(conn.to_skill))?;
            
            // Check target input exists
            if !to_skill.has_input(&conn.to_input) {
                return Err(ComposerError::InputNotFound(
                    conn.to_skill, 
                    conn.to_input.clone()
                ));
            }
        }
        
        Ok(())
    }
    
    fn detect_cycles(&self) -> Result<(), ComposerError> {
        // Build dependency graph and check for cycles
        // Implementation using topological sort
        Ok(())
    }
    
    fn remap_hash(skill_hash: &ContentHash, node_hash: &NodeHash) -> NodeHash {
        // Create unique hash combining skill and node
        let mut hasher = sha2::Sha256::new();
        hasher.update(&skill_hash.0);
        hasher.update(&node_hash.0);
        NodeHash(hasher.finalize().into())
    }
    
    fn remap_node(
        &self,
        node: &RuntimeNode,
        skill_hash: &ContentHash,
        mapping: &HashMap<(ContentHash, NodeHash), NodeHash>,
    ) -> Result<RuntimeNode, ComposerError> {
        // Clone node with remapped input hashes
        Ok(node.clone())
    }
    
    fn create_bridge_node(
        &self,
        connection: &SkillConnection,
        mapping: &HashMap<(ContentHash, NodeHash), NodeHash>,
    ) -> Result<RuntimeNode, ComposerError> {
        // Create a node that bridges two skills
        Ok(RuntimeNode::Operation {
            op: zerolang::Op::Identity,
            inputs: Vec::new(),
        })
    }
}

#[derive(Debug)]
pub enum ComposerError {
    SkillNotFound(ContentHash),
    OutputNotFound(ContentHash, String),
    InputNotFound(ContentHash, String),
    CycleDetected,
    GraphBuildError(String),
}
```

## Skill Verifier

**File:** `src/skills/verifier.rs`

```rust
use zerolang::RuntimeGraph;

/// Verify skill graphs are safe to execute
pub struct SkillVerifier;

#[derive(Debug)]
pub struct VerificationResult {
    pub safe: bool,
    pub warnings: Vec<VerificationWarning>,
    pub errors: Vec<VerificationError>,
    pub proof: Option<SafetyProof>,
}

#[derive(Debug)]
pub struct SafetyProof {
    pub halting_proven: bool,
    pub max_steps: Option<u64>,
    pub memory_bound: Option<u64>,
    pub no_external_calls: bool,
}

#[derive(Debug)]
pub enum VerificationWarning {
    LargeGraph { node_count: usize },
    UnboundedLoop { node_hash: [u8; 32] },
    ExternalCall { uri: String },
}

#[derive(Debug)]
pub enum VerificationError {
    InfiniteLoop { cycle: Vec<[u8; 32]> },
    UnsafeOperation { op: String, reason: String },
    MissingInput { input_name: String },
    TypeMismatch { expected: String, found: String },
}

impl SkillVerifier {
    /// Verify a skill graph is safe to execute
    pub fn verify(graph: &RuntimeGraph) -> Result<VerificationResult, SkillError> {
        let mut warnings = Vec::new();
        let mut errors = Vec::new();
        
        // 1. Check graph size
        let node_count = graph.nodes().len();
        if node_count > 1000 {
            warnings.push(VerificationWarning::LargeGraph { node_count });
        }
        
        // 2. Verify halting (no infinite loops)
        if let Some(cycle) = Self::find_cycle(graph) {
            errors.push(VerificationError::InfiniteLoop { cycle });
        }
        
        // 3. Check for unsafe operations
        for node in graph.nodes() {
            if let Some(error) = Self::check_node_safety(node) {
                errors.push(error);
            }
        }
        
        // 4. Verify type consistency
        for (expected, found) in Self::check_types(graph) {
            errors.push(VerificationError::TypeMismatch { expected, found });
        }
        
        // 5. Check external calls
        let external_uris = Self::find_external_calls(graph);
        for uri in &external_uris {
            warnings.push(VerificationWarning::ExternalCall { uri: uri.clone() });
        }
        
        // 6. Build safety proof
        let proof = if errors.is_empty() {
            Some(SafetyProof {
                halting_proven: Self::prove_halting(graph).is_ok(),
                max_steps: Self::estimate_max_steps(graph),
                memory_bound: Self::estimate_memory_bound(graph),
                no_external_calls: external_uris.is_empty(),
            })
        } else {
            None
        };
        
        Ok(VerificationResult {
            safe: errors.is_empty(),
            warnings,
            errors,
            proof,
        })
    }
    
    fn find_cycle(graph: &RuntimeGraph) -> Option<Vec<[u8; 32]>> {
        // Use DFS to find cycles
        None
    }
    
    fn check_node_safety(node: &RuntimeNode) -> Option<VerificationError> {
        // Check if node operation is safe
        None
    }
    
    fn check_types(graph: &RuntimeGraph) -> Vec<(String, String)> {
        // Type check all connections
        Vec::new()
    }
    
    fn find_external_calls(graph: &RuntimeGraph) -> Vec<String> {
        // Find all External nodes and return their URIs
        Vec::new()
    }
    
    fn prove_halting(graph: &RuntimeGraph) -> Result<(), ()> {
        // Try to prove the graph halts
        Ok(())
    }
    
    fn estimate_max_steps(graph: &RuntimeGraph) -> Option<u64> {
        // Estimate maximum execution steps
        Some(graph.nodes().len() as u64 * 10)
    }
    
    fn estimate_memory_bound(graph: &RuntimeGraph) -> Option<u64> {
        // Estimate maximum memory usage
        Some(1024 * 1024) // 1MB default
    }
}
```

## Built-in Skills

### Echo Skill

**File:** `src/skills/builtin/echo.rs`

```rust
use zerolang::{RuntimeGraph, RuntimeNode, Op};

/// Create the echo skill graph
pub fn create_echo_skill() -> RuntimeGraph {
    RuntimeGraph::builder()
        .name("echo")
        .description("Echoes the input message back")
        .add_node(RuntimeNode::External { 
            uri: "input://message".to_string(),
            inputs: vec![],
        })
        .add_node(RuntimeNode::Operation {
            op: Op::Identity,
            inputs: vec![/* input node hash */],
        })
        .output("output")
        .build()
        .expect("Echo skill should build successfully")
}
```

**Graph File:** `graphs/skills/echo.0`

```
# Echo Skill
#
# Simply returns the input message.
# Useful for testing and as a template for other skills.
#
# Inputs:
#   - message: StringTensor containing the message to echo
#
# Outputs:
#   - response: StringTensor containing the echoed message

Graph {
    name: "echo",
    version: 1,
    description: "Echoes input back to output",
    
    nodes: [
        { id: "input", type: External, uri: "input://message" },
        { id: "format", type: Operation, op: StringFormat, 
          inputs: ["input"], 
          template: "Echo: {}" 
        },
        { id: "output", type: Operation, op: Identity, inputs: ["format"] },
    ],
    
    entry_point: "input",
    outputs: ["output"],
    
    proofs: [
        HaltingProof { max_steps: 3, fuel_budget: 100 }
    ]
}
```

### Search Skill

**File:** `graphs/skills/search.0`

```
# Search Skill
#
# Performs web search and returns results.
#
# Inputs:
#   - query: StringTensor containing search query
#
# Outputs:
#   - results: StringTensor containing formatted search results

Graph {
    name: "search",
    version: 1,
    description: "Web search skill",
    
    nodes: [
        { id: "query", type: External, uri: "input://query" },
        
        # Call external search API
        { id: "search_api", type: External, 
          uri: "https://api.search.example/search",
          inputs: ["query"]
        },
        
        # Parse JSON response
        { id: "parse", type: Operation, op: JsonParse, inputs: ["search_api"] },
        
        # Extract results
        { id: "results", type: Operation, op: JsonGet, 
          inputs: ["parse"], 
          path: "results" 
        },
        
        # Format output
        { id: "format", type: Operation, op: StringFormat,
          inputs: ["results"],
          template: "Search Results:\n{}"
        },
    ],
    
    outputs: ["format"],
    permissions: ["network"],
}
```

## Getting Started

```bash
cd /Users/JiahaoRBC/Git/0-protocol/0-openclaw
git checkout -b agent9/skills

# Wait for Agent #0 and Agent #7, then:
cat src/lib.rs
cat src/gateway/mod.rs

# Create skills module
mkdir -p src/skills/builtin
touch src/skills/mod.rs
touch src/skills/registry.rs
touch src/skills/composer.rs
touch src/skills/verifier.rs
touch src/skills/loader.rs

# Create skill graphs
mkdir -p graphs/skills
touch graphs/skills/echo.0
touch graphs/skills/search.0
touch graphs/skills/browser.0
```

## Commit Format

```
[Agent#9] feat: Implement SkillRegistry with content-addressed storage
[Agent#9] feat: Add SkillComposer for workflow composition
[Agent#9] feat: Implement SkillVerifier with safety proofs
[Agent#9] feat: Create echo skill graph
[Agent#9] feat: Create search skill graph
[Agent#9] feat: Add browser skill for web automation
```

## Success Criteria

1. SkillRegistry stores and retrieves skills correctly
2. Skills are content-addressed (same content = same hash)
3. SkillComposer creates valid composed graphs
4. SkillVerifier catches unsafe operations
5. Built-in skills execute correctly
6. Verification produces valid safety proofs

## Dependencies on Other Agents

- **Agent #0**: Need repository structure and types
- **Agent #6**: Need 0-lang graph operations (StringFormat, JsonParse, etc.)
- **Agent #7**: Need Gateway to register skills with

## Handoff Points

When you complete:
- **SkillRegistry**: Notify Agent #7 (gateway needs to load skills)
- **SkillComposer**: Advanced skill workflows enabled
- **Built-in skills**: Notify Agent #10 (CLI can test skills)
- **SkillVerifier**: Security audits can begin

## Questions?

Document blockers in `BLOCKERS.md` in your branch.
