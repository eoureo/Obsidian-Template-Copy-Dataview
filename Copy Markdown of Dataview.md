<%_*
/*
Obsidian Templater template that copies Obsidian Dataview results to clipboard as Markdown text.

The cursor(caret) must be located inside the Dataview codeblock in Editing View(Source or Live Preview). 
Then run this template.

eoureo
@2023-05-01 20:00:00
*/
const { Notice } = tp.obsidian;

// 에디터가 활성화 되어 커서 있는 곳의 소스가 보이도록 기다립니다.
app.workspace.activeLeaf.view.editor.focus();

const sleep_count_limit = 10; // 코드 블럭이 선택 될 때까지 최대 10번 반복합니다.

let line_cursor;
let lines;
let line_num_cursor;
let line_num_start;
let line_num_end;
let sleep_count;

document.body.style.cursor  = 'wait';
for(sleep_count = 0; sleep_count < sleep_count_limit; sleep_count++) {
  line_cursor = document.querySelector(".workspace-leaf.mod-active .cm-active");
  
  try {
    lines = Array.from(document.querySelectorAll(".workspace-leaf.mod-active .cm-line"));
    // 커서 위치 줄
    line_num_cursor = lines.indexOf(line_cursor);
    // 코드 블럭 시작 줄
    line_num_start = find_line_startwith("```", line_num_cursor, direction = -1);
    // 코드 블럭 끝 줄
    line_num_end = find_line_startwith("```", line_num_cursor, direction = 1);

    if(line_cursor.classList.contains("HyperMD-codeblock") && line_num_start >= 0 && line_num_end >= 0 && line_num_start < line_num_end) {
      break; // 코드 블럭 찾으면 반복 끝냄
    }
  }
  catch(e){} // 줄 찾을 때 나오는 에러 무시
  await sleep(300); // 0.3초 기다림
}
document.body.style.cursor  = 'default';
// console.log("sleep_count:", sleep_count);

if(!line_cursor.classList.contains("HyperMD-codeblock")) {
  notice("에러! 커서가 코드 블럭 안에 있어야 합니다.");
  return;
}

if(line_num_start < 0 || line_num_end < 0 || line_num_start >= line_num_end) {
  notice(`에러! 커서가 코드 블럭 안에 있어야 합니다.\n(${sleep_count} , ${line_num_start}:${line_num_cursor}:${line_num_end})`);
  return;
}

const dv = app.plugins.plugins["dataview"].api; // dataview 사용

const dv_stack = []; // 데이터뷰 결과들을 저장

const codeblock_lines = [];
for(let line_num = line_num_start + 1; line_num < line_num_end; line_num++) {
  codeblock_lines.push(lines[line_num].textContent.trim());
}

// for checking if caller of function is tp root
function get_caller(){
  return get_caller.caller;
}
const caller = get_caller();

// dataviewjs 코드 블럭인지 dataview 코드 블럭인지에 따라 다르게 실행
if(/^`{1,}dataviewjs/.test(lines[line_num_start].textContent.trim())) {
  // dataviewjs 코드 블럭
  let codeblock_string = codeblock_lines.join("\n").replaceAll(";.", ".").replace(/\u200B/g,'');

  //  Rendering Functions
  function dv_table (...args) {
    let md = dv.markdownTable(...args);
    if(dv_table.caller == caller) {
      dv_stack.push(md + "\n");
    }
    else {
      return md;
    }
  }
  function dv_list (...args) {
    let md = dv.markdownList(...args);
    if(dv_list.caller == caller) {
      dv_stack.push(md + "\n");
    }
    else {
      return md;
    }
  }
  function dv_taskList (...args) {
    let md = dv.markdownTaskList(...args);
    if(dv_taskList.caller == caller) {
      dv_stack.push(md + "\n");
    }
    else {
      return md;
    }
  }
  function dv_el (element, text, info) {
    let elm = document.createElement(element);
    elm.innerHTML = text;
    if(info?.attr) {
      for(let attr_key in info.attr) {
        elm.setAttribute(attr_key, info.attr[attr_key]);
      }
    }
    if(info?.cls) {
    elm.setAttribute("class", info.cls);
    }
    
    if(dv_el.caller == caller) {
      dv_stack.push(elm.outerHTML);
    }
    else {
      return elm.outerHTML;
    }
  }
  function dv_header (level, text) {
    let md = "#".repeat(level) + " " + text + "\n";
    if(dv_header.caller == caller) {
      dv_stack.push(md + "\n");
    }
    else {
      return md;
    }
  }
  function dv_span (text) {
    if(dv_span.caller == caller) {
      dv_stack.push(text);
    }
    else {
      return text;
    }
  }
  function dv_paragraph (text) {
    let md = "\n\n" + text + "\n\n";
    if(dv_paragraph.caller == caller) {
      dv_stack.push(md);
    }
    else {
      return md;
    }
  }
  async function dv_execute (text) {
    let result = await dv.queryMarkdown(text);
    if(result?.value) {
      // if(dv_execute.caller == caller) {
        dv_stack.push(result.value + "\n");
      // }
      // else {
      //   return result.value;
      // }
    }
  }
  
  codeblock_string = codeblock_string.replace(/dv\.table\s*\(/g, "dv_table(")
                      .replace(/dv\.list\s*\(/g, "dv_list(")
                      .replace(/dv\.taskList\s*\(/g, "dv_taskList(")
                      .replace(/dv\.header\s*\(/g, "dv_header(")
                      .replace(/dv\.el\s*\(/g, "dv_el(")
                      .replace(/dv\.span\s*\(/g, "dv_span(")
                      .replace(/dv\.paragraph\s*\(/g, "dv_paragraph(")
                      .replace(/dv\.executeJs\s*\(/g, "eval(")
                      .replace(/dv\.execute\s*\(/g, "dv_execute(");
  
  await eval(codeblock_string);
}
else if(/^`{1,}dataview/.test(lines[line_num_start].textContent.trim())) {
  // dataview 코드 블럭
  let codeblock_string = codeblock_lines.join("\n").replace(/\u200B/g,'');
  console.log(codeblock_string);

  // List, Task만 있으면 에러가 남. 마지막에 공란을 넣으면 해결 됨.
  let result = await dv.queryMarkdown(codeblock_string + " ");
  
  if(result?.value) {
    dv_stack.push(result.value);
  }
}
else {
  notice(`dataview 코드 블럭이 아닙니다.`);
  return;
}
console.log(dv_stack);

let content = dv_stack.join("");
console.log(content);

let message;
if(content != "") {
  navigator.clipboard.writeText(content).then(()=> {
    notice(`마크다운 형식으로 클립보드에 복사 되었습니다.\n이제 붙여넣기를 할 수 있습니다.`);
  },() => {
    notice(`복사 되지 않았습니다.`);
  });
  return;
}
else {
  notice(`복사할 내용이 없습니다.`);
}

function find_line_startwith(sting_start, line_num, direction = 1) {
  if(line_num < 0) {
    return -1;
  }
  
  let line = lines[line_num];
  if(!line) {
    return -1;
  }
  
  if(line.textContent.startsWith(sting_start)) {
    return line_num;
  }
  else {
    return find_line_startwith(sting_start, line_num + direction, direction);
  }
}

function notice(message) {
  new Notice(message);
  console.log(message);
}

return;
_%>
