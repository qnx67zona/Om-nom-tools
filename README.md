# Om-nom-tools
const fs = require('fs');
const path = require('path');
const dox = require('dox');

class DocParser {
  constructor(srcDir, outDir) {
    this.srcDir = srcDir;
    this.outDir = outDir;
    this.files = [];
    this.documentation = [];
    this.config = {
      title: 'API Documentation',
      version: '1.0.0',
      showPrivate: false,
      sortByName: true
    };
  }

  readFiles() {
    try {
      const items = fs.readdirSync(this.srcDir);
      this.files = items
        .filter(f => f.endsWith('.js') && !f.startsWith('.'))
        .map(f => path.join(this.srcDir, f));
      
      console.log(`Found ${this.files.length} files to process`);
    } catch(err) {
      console.error(`Error reading directory: ${err.message}`);
      process.exit(1);
    }
  }

  parse() {
    let allDocs = [];
    
    this.files.forEach(file => {
      try {
        const src = fs.readFileSync(file, 'utf8');
        const filename = path.basename(file);
        
        const parsed = dox.parseCode(src);
        
        const filtered = parsed.filter(doc => {
          if (!doc.ctx) return false;
          if (this.config.showPrivate === false && doc.isPrivate) return false;
          return true;
        });

        if (filtered.length > 0) {
          allDocs.push({ 
            file: filename, 
            path: file,
            docs: filtered,
            count: filtered.length
          });
        }
      } catch(e) {
        console.error(`Error parsing ${filename}: ${e.message}`);
      }
    });

    if (this.config.sortByName) {
      allDocs.forEach(item => {
        item.docs.sort((a, b) => {
          if (!a.ctx || !b.ctx) return 0;
          return a.ctx.name.localeCompare(b.ctx.name);
        });
      });
    }

    this.documentation = allDocs;
    console.log(`Parsed ${allDocs.length} files with documentation`);
    return allDocs;
  }

  formatType(types) {
    if (!types || !Array.isArray(types)) return 'any';
    return types.join(' | ');
  }

  formatTags(tags) {
    if (!tags || !tags.length) return '';
    
    let html = '';
    const params = tags.filter(t => t.type === 'param');
    const returns = tags.filter(t => t.type === 'return' || t.type === 'returns');
    const deprecated = tags.find(t => t.type === 'deprecated');
    const example = tags.find(t => t.type === 'example');

    if (params.length) {
      html += '<div class="params-section">\n';
      html += '<h4>Parameters</h4>\n';
      html += '<table class="params-table">\n';
      html += '<thead><tr><th>Name</th><th>Type</th><th>Description</th></tr></thead>\n';
      html += '<tbody>\n';
      
      params.forEach(p => {
        const type = this.formatType(p.types);
        html += `<tr><td><code>${p.name}</code></td><td><code>${type}</code></td><td>${p.description || '-'}</td></tr>\n`;
      });
      
      html += '</tbody>\n</table>\n</div>\n';
    }

    if (returns.length) {
      html += '<div class="returns-section">\n';
      html += '<h4>Returns</h4>\n';
      returns.forEach(r => {
        const type = this.formatType(r.types);
        html += `<p><strong>Type:</strong> <code>${type}</code></p>\n`;
        if (r.description) {
          html += `<p>${r.description}</p>\n`;
        }
      });
      html += '</div>\n';
    }

    if (deprecated) {
      html += `<div class="warning">⚠️ <strong>Deprecated:</strong> ${deprecated.description}</div>\n`;
    }

    if (example) {
      html += '<div class="example-section">\n';
      html += '<h4>Example</h4>\n';
      html += `<pre><code>${this.escapeHTML(example.string)}</code></pre>\n`;
      html += '</div>\n';
    }

    return html;
  }

  escapeHTML(str) {
    if (!str) return '';
    return str
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;')
      .replace(/"/g, '&quot;')
      .replace(/'/g, '&#039;');
  }

  generateFunctionHTML(func) {
    if (!func.ctx) return '';

    let html = '<div class="function-doc">\n';
    
    const visibility = func.isPrivate ? '(private) ' : '';
    const type = func.ctx.type || 'function';
    
    html += `<h3><span class="type">${type}</span> ${visibility}${func.ctx.name}</h3>\n`;
    
    if (func.ctx.string) {
      html += `<div class="signature"><code>${this.escapeHTML(func.ctx.string)}</code></div>\n`;
    }

    if (func.description && func.description.full) {
      html += `<div class="description">${func.description.full}</div>\n`;
    }

    if (func.description && func.description.summary) {
      html += `<div class="summary">${func.description.summary}</div>\n`;
    }

    html += this.formatTags(func.tags);

    html += '</div>\n';
    return html;
  }

  writeHTML(docs) {
    let html = this.getTemplate();

    const toc = this.generateTableOfContents(docs);
    html += toc;

    html += '<div class="content">\n';

    docs.forEach((item, idx) => {
      html += `<div class="file-section" id="file-${idx}">\n`;
      html += `<h2>${item.file}</h2>\n`;
      html += `<p class="file-info">File contains ${item.count} documented function${item.count !== 1 ? 's' : ''}</p>\n`;
      
      item.docs.forEach((doc, funcIdx) => {
        html += `<div class="function-wrapper" id="func-${idx}-${funcIdx}">\n`;
        html += this.generateFunctionHTML(doc);
        html += '</div>\n';
      });

      html += '</div>\n';
    });

    html += '</div>\n';
    html += this.getFooter();
    html += '</body></html>\n';

    if (!fs.existsSync(this.outDir)) {
      fs.mkdirSync(this.outDir, { recursive: true });
    }

    const outputPath = path.join(this.outDir, 'index.html');
    fs.writeFileSync(outputPath, html, 'utf8');
    console.log(`Documentation written to ${outputPath}`);
  }

  generateTableOfContents(docs) {
    let html = '<div class="toc">\n';
    html += '<h2>Contents</h2>\n';
    html += '<ul class="toc-list">\n';

    docs.forEach((item, idx) => {
      html += `<li><a href="#file-${idx}">${item.file}</a>\n`;
      html += '<ul class="toc-sublist">\n';
      
      item.docs.forEach((doc, funcIdx) => {
        if (doc.ctx) {
          html += `<li><a href="#func-${idx}-${funcIdx}">${doc.ctx.name}</a></li>\n`;
        }
      });

      html += '</ul>\n</li>\n';
    });

    html += '</ul>\n</div>\n';
    return html;
  }

  getTemplate() {
    return `<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>${this.config.title}</title>
<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
  line-height: 1.6;
  color: #333;
  background: #f5f5f5;
}

header {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  padding: 40px 20px;
  text-align: center;
  box-shadow: 0 2px 10px rgba(0,0,0,0.1);
}

header h1 {
  font-size: 2.5em;
  margin-bottom: 10px;
}

header .version {
  font-size: 0.9em;
  opacity: 0.9;
}

.container {
  display: flex;
  max-width: 1400px;
  margin: 0 auto;
}

.toc {
  width: 250px;
  background: white;
  padding: 30px 20px;
  border-right: 1px solid #ddd;
  position: sticky;
  top: 0;
  height: 100vh;
  overflow-y: auto;
  box-shadow: 2px 0 5px rgba(0,0,0,0.05);
}

.toc h2 {
  font-size: 1.2em;
  margin-bottom: 20px;
  color: #667eea;
}

.toc-list, .toc-sublist {
  list-style: none;
}

.toc-list > li {
  margin-bottom: 15px;
}

.toc-list a {
  color: #333;
  text-decoration: none;
  font-weight: 600;
  display: block;
  padding: 5px 0;
  transition: color 0.2s;
}

.toc-list a:hover {
  color: #667eea;
}

.toc-sublist {
  margin-left: 15px;
  margin-top: 8px;
  border-left: 2px solid #e0e0e0;
  padding-left: 10px;
}

.toc-sublist li {
  margin: 5px 0;
}

.toc-sublist a {
  font-weight: 400;
  font-size: 0.9em;
  color: #666;
}

.toc-sublist a:hover {
  color: #764ba2;
}

.content {
  flex: 1;
  padding: 40px;
}

.file-section {
  background: white;
  padding: 30px;
  margin-bottom: 30px;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.file-section h2 {
  color: #667eea;
  font-size: 1.8em;
  margin-bottom: 10px;
  
