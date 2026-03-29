Great — now that Rojo works, here are the **best VS Code plugins for a modern Roblox + Rojo workflow**.  
These are the same tools used by most professional Roblox developers working outside Studio.

---

# ✅ **1. Roblox LSP (Roblox Lua Language Server)**

**⭐ MUST-HAVE — the most important extension**

Provides:

- Autocomplete for Roblox APIs
    
- Type checking (Luau types)
    
- IntelliSense for Instance trees
    
- Errors/warnings inline
    
- Hover documentation
    
- Workspace symbol search
    

This is what makes VS Code behave like a real Roblox editor.

**VS Code marketplace:**  
`Roblox LSP` (by Roblox)

---

# ✅ **2. Rojo (Official VS Code Extension)**

Adds:

- Rojo project file detection
    
- Button to quickly launch `rojo serve`
    
- Syntax helpers for `.project.json`
    
- Status bar integration
    

This doesn’t replace the CLI, but makes your workflow smoother with Studio + Rojo.

---

# ✅ **3. Stylua (Formatter)**

Auto-formats your code every save.

Why use it:

- Clean, consistent Luau style
    
- Fixes spacing, indentation
    
- Works with Luau type annotations
    

Install:

- VS Code extension: `Stylua`
    
- And install CLI:
    
    ```bash
    cargo install stylua
    ```
    

Then add a `.stylua.toml` config in your repo.

---

# ✅ **4. Luau Syntax Highlighting**

If you want better highlighting or themes.

Extensions:

- `Luau Language Support`
    
- `Luau Syntax`
    

Roblox LSP includes most syntax support, but some developers like more colorful highlighting.

---

# ➕ OPTIONAL (but highly recommended for larger projects)

## **5. GitLens**

Super strong Git UI for tracking changes — vital for Rojo workflows.

## **6. Todo Tree**

Scans comments for `TODO`, `FIXME`, `NOTE`, etc.  
Helpful when working across many scripts.

## **7. Error Lens**

Shows errors and warnings inline (pairs great with Rojo + LSP).

---

# 🎯 Recommended Setup Order

1. **Roblox LSP**
    
2. **Rojo VS Code extension**
    
3. **Stylua**
    
4. **Luau Syntax (optional)**
    
5. Extra QOL extensions as needed
    

---

# 📁 If you want, I can also generate for you:

- A clean **Rojo project template**
    
- A ready-to-use `default.project.json`
    
- Pre-made `.stylua.toml`
    
- Folder structure for **Server**, **Client**, and **Shared** directories
    

Just tell me:  
**Do you want me to generate the template?**