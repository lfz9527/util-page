# 身份证生成器 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建一个单文件 HTML 身份证数据模拟工具，支持性别/出生日期/省市区三级联动选择，生成符合 GB 11643 规范的合法虚拟身份证信息。

**Architecture:** 单文件 `idcard-generator.html`，内嵌 `<style>` 和 `<script>`。页面分三个区域：顶部控制面板（表单控件）、中部结果卡片（信息列表+复制按钮）、底部 JS 数据常量区。无外部依赖，浏览器直接打开即用。

**Tech Stack:** HTML5 + CSS3 + Vanilla JS (ES6)

## Global Constraints

- 单文件 `idcard-generator.html`，零外部依赖
- 身份证号必须符合 GB 11643 校验码规则
- 身份证号第 17 位奇偶必须与性别对应
- 地区三级联动：选省→刷新市→选市→刷新区
- 日期选择器：日随年月动态变化，正确处理闰年
- 首次加载预填默认值并自动生成一次结果

---

### Task 1: HTML 骨架 + CSS 样式

**Files:**
- Create: `idcard-generator.html`

**Interfaces:**
- Produces: HTML 元素 ID — `#gender`, `#year`, `#month`, `#day`, `#province`, `#city`, `#district`, `#btn-generate`, `#result-name`, `#result-gender`, `#result-ethnicity`, `#result-birth`, `#result-address`, `#result-id`, `#btn-copy`, `#hint-address`

- [ ] **Step 1: 创建文件，写入 HTML 骨架和 CSS 样式**

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>身份证生成器</title>
<style>
* { margin: 0; padding: 0; box-sizing: border-box; }
body {
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", "PingFang SC", "Microsoft YaHei", sans-serif;
  background: #f0f2f5;
  min-height: 100vh;
  display: flex;
  justify-content: center;
  padding: 40px 16px;
}
.container { width: 100%; max-width: 480px; }

/* 卡片通用 */
.card {
  background: #fff;
  border-radius: 12px;
  padding: 28px 24px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.08);
  margin-bottom: 20px;
}
.card-title {
  font-size: 16px;
  font-weight: 600;
  color: #1a1a1a;
  margin-bottom: 20px;
  padding-bottom: 12px;
  border-bottom: 1px solid #f0f0f0;
}

/* 表单行 */
.form-row {
  display: flex;
  align-items: center;
  margin-bottom: 16px;
}
.form-row:last-child { margin-bottom: 0; }
.form-label {
  width: 72px;
  flex-shrink: 0;
  font-size: 14px;
  color: #666;
}
.form-select, .form-input {
  flex: 1;
  height: 38px;
  padding: 0 12px;
  border: 1px solid #d9d9d9;
  border-radius: 6px;
  font-size: 14px;
  color: #333;
  background: #fff;
  cursor: pointer;
  outline: none;
  transition: border-color 0.2s;
}
.form-select:focus, .form-input:focus { border-color: #1677ff; }
.form-select:disabled {
  background: #f5f5f5;
  color: #bbb;
  cursor: not-allowed;
}
.form-select + .form-select { margin-left: 8px; }

/* 生成按钮 */
.btn-generate {
  width: 100%;
  height: 44px;
  margin-top: 20px;
  border: none;
  border-radius: 8px;
  font-size: 16px;
  font-weight: 500;
  color: #fff;
  background: #1677ff;
  cursor: pointer;
  transition: background 0.2s;
}
.btn-generate:hover { background: #4096ff; }
.btn-generate:disabled {
  background: #d9d9d9;
  color: #999;
  cursor: not-allowed;
}
.hint-address {
  font-size: 12px;
  color: #ff4d4f;
  margin-top: 8px;
  display: none;
}
.hint-address.visible { display: block; }

/* 结果卡片 */
.result-item {
  display: flex;
  align-items: center;
  padding: 12px 0;
  border-bottom: 1px solid #f5f5f5;
}
.result-item:last-child { border-bottom: none; }
.result-label {
  width: 72px;
  flex-shrink: 0;
  font-size: 14px;
  color: #999;
}
.result-value {
  flex: 1;
  font-size: 15px;
  color: #1a1a1a;
  font-weight: 500;
}
.result-value.id-number {
  font-family: "SF Mono", "Menlo", "Consolas", monospace;
  letter-spacing: 2px;
  font-size: 17px;
}

/* 复制按钮 */
.btn-copy {
  flex-shrink: 0;
  padding: 4px 12px;
  font-size: 12px;
  color: #1677ff;
  background: #e6f4ff;
  border: 1px solid #91caff;
  border-radius: 4px;
  cursor: pointer;
  margin-left: 8px;
  transition: all 0.2s;
}
.btn-copy:hover { background: #bae0ff; }
.btn-copy.copied {
  color: #52c41a;
  background: #f6ffed;
  border-color: #b7eb8f;
}

/* 空状态 */
.result-empty {
  text-align: center;
  color: #bbb;
  padding: 24px 0;
  font-size: 14px;
}
</style>
</head>
<body>
<div class="container">

  <!-- 控制面板 -->
  <div class="card">
    <div class="card-title">选项设置</div>

    <div class="form-row">
      <span class="form-label">性别</span>
      <select id="gender" class="form-select">
        <option value="male">男</option>
        <option value="female">女</option>
      </select>
    </div>

    <div class="form-row">
      <span class="form-label">出生日期</span>
      <select id="year" class="form-select"></select>
      <select id="month" class="form-select"></select>
      <select id="day" class="form-select"></select>
    </div>

    <div class="form-row">
      <span class="form-label">居住地址</span>
      <select id="province" class="form-select"><option value="">请选择省</option></select>
      <select id="city" class="form-select" disabled><option value="">请选择市</option></select>
      <select id="district" class="form-select" disabled><option value="">请选择区</option></select>
    </div>

    <button id="btn-generate" class="btn-generate" disabled>生成身份证</button>
    <p id="hint-address" class="hint-address">请先选择完整地址</p>
  </div>

  <!-- 结果卡片 -->
  <div class="card" id="result-card">
    <div class="card-title">生成结果</div>
    <div id="result-container">
      <div class="result-empty">点击上方「生成身份证」按钮查看结果</div>
    </div>
  </div>

</div>
</body>
</html>
```

- [ ] **Step 2: 浏览器打开验证页面布局**

打开 `idcard-generator.html`，确认三个卡片区域正常渲染，表单控件布局正确。

---

### Task 2: JavaScript 数据层

**Files:**
- Modify: `idcard-generator.html` — 在 `</body>` 前插入 `<script>` 标签及数据常量

**Interfaces:**
- Produces:
  - `const REGION_DATA` — 省市区三级嵌套对象，结构: `{ "省名": { code: "xxxxxx", cities: { "市名": { code: "xxxxxx", districts: { "区名": "xxxxxx" } } } } }`
  - `const SURNAMES` — `string[]`，约 100 个常见姓氏
  - `const MALE_NAMES` — `string[]`，约 150 个男性常用名字字
  - `const FEMALE_NAMES` — `string[]`，约 150 个女性常用名字字

- [ ] **Step 1: 在 `</body>` 前插入 `<script>` 标签，写入地区数据**

在 `</body>` 前添加：

```html
<script>
/* ========== 地区数据 ========== */
const REGION_DATA = {
  "北京市": {
    code: "110000",
    cities: {
      "市辖区": {
        code: "110100",
        districts: {
          "东城区": "110101", "西城区": "110102", "朝阳区": "110105",
          "丰台区": "110106", "石景山区": "110107", "海淀区": "110108",
          "门头沟区": "110109", "房山区": "110111", "通州区": "110112",
          "顺义区": "110113", "昌平区": "110114", "大兴区": "110115",
          "怀柔区": "110116", "平谷区": "110117", "密云区": "110118",
          "延庆区": "110119"
        }
      }
    }
  },
  "天津市": {
    code: "120000",
    cities: {
      "市辖区": {
        code: "120100",
        districts: {
          "和平区": "120101", "河东区": "120102", "河西区": "120103",
          "南开区": "120104", "河北区": "120105", "红桥区": "120106",
          "东丽区": "120110", "西青区": "120111", "津南区": "120112",
          "北辰区": "120113", "武清区": "120114", "宝坻区": "120115",
          "滨海新区": "120116", "宁河区": "120117", "静海区": "120118",
          "蓟州区": "120119"
        }
      }
    }
  },
  "河北省": {
    code: "130000",
    cities: {
      "石家庄市": {
        code: "130100",
        districts: {
          "长安区": "130102", "桥西区": "130104", "新华区": "130105",
          "井陉矿区": "130107", "裕华区": "130108", "藁城区": "130109",
          "鹿泉区": "130110", "栾城区": "130111", "井陉县": "130121",
          "正定县": "130123", "行唐县": "130125", "灵寿县": "130126",
          "高邑县": "130127", "深泽县": "130128", "赞皇县": "130129",
          "无极县": "130130", "平山县": "130131", "元氏县": "130132",
          "赵县": "130133", "辛集市": "130181", "晋州市": "130183",
          "新乐市": "130184"
        }
      },
      "唐山市": {
        code: "130200",
        districts: {
          "路南区": "130202", "路北区": "130203", "古冶区": "130204",
          "开平区": "130205", "丰南区": "130207", "丰润区": "130208",
          "曹妃甸区": "130209", "滦南县": "130224", "乐亭县": "130225",
          "迁西县": "130227", "玉田县": "130229", "遵化市": "130281",
          "迁安市": "130283", "滦州市": "130284"
        }
      }
      /* 其余城市按同样结构补齐：秦皇岛市、邯郸市、邢台市、保定市、张家口市、承德市、沧州市、廊坊市、衡水市 */
    }
  }
  /* 其余省份按同样结构补齐：山西省、内蒙古自治区、辽宁省、吉林省、黑龙江省、
     上海市、江苏省、浙江省、安徽省、福建省、江西省、山东省、河南省、湖北省、
     湖南省、广东省、广西壮族自治区、海南省、重庆市、四川省、贵州省、云南省、
     西藏自治区、陕西省、甘肃省、青海省、宁夏回族自治区、新疆维吾尔自治区 */
};
</script>
```

> **实现时须补齐**：以上仅展示 3 个省的结构示例，实现时需补齐全部 31 个省级行政区的完整区县数据。

- [ ] **Step 2: 写入姓名库数据**

紧接地区数据之后：

```js
/* ========== 姓名库 ========== */
const SURNAMES = [
  "王","李","张","刘","陈","杨","黄","赵","周","吴",
  "徐","孙","马","胡","朱","郭","何","罗","高","林",
  "郑","梁","谢","唐","许","冯","宋","韩","邓","彭",
  "曹","曾","田","于","萧","潘","袁","蔡","蒋","余",
  "杜","叶","程","苏","魏","吕","丁","任","卢","姚",
  "沈","钟","姜","崔","谭","陆","范","汪","廖","石",
  "金","韦","贾","夏","付","方","白","邹","孟","熊",
  "秦","邱","江","尹","薛","闫","段","雷","侯","龙",
  "史","陶","黎","贺","顾","毛","郝","龚","邵","万",
  "钱","严","覃","武","戴","莫","孔","向","汤","温"
];

const MALE_NAMES = [
  "伟","强","磊","军","勇","文","明","杰","涛","斌",
  "辉","鹏","浩","峰","亮","超","波","刚","健","毅",
  "俊","飞","宁","帅","旭","华","林","平","东","志",
  "宏","海","建","国","宇","龙","博","松","瑞","凯",
  "鑫","震","恒","义","清","哲","康","良","威","仁",
  "云","诚","安","忠","庆","彬","富","信","光","天",
  "达","广","生","思","厚","友","维","全","保","世",
  "兴","源","和","延","士","福","之","道","德","山",
  "川","根","柱","喜","立","永","才","正","玉","春",
  "秋","炎","盛","雄","善","舜","谦","豪","英","朗",
  "锐","腾","景","仕","易","知","合","柏","腾","乾",
  "启","承","修","铭","泽","恩","冠","宸","敬","贤",
  "尚","尊","驰","鸣","弘","绍","庭","晋","卫","守",
  "栩","颢","瑾","玮","骏","邦","颂","祯","琨","敖",
  "晏","长","沛","宣","赫","翊","昭","朔","麒","腾"
];

const FEMALE_NAMES = [
  "芳","秀","英","丽","敏","静","婷","雪","娟","艳",
  "萍","红","霞","玲","娜","燕","琴","文","梅","云",
  "华","兰","凤","洁","春","慧","佳","蕾","莉","晶",
  "蓉","丹","芬","颖","欣","雨","瑶","倩","月","媛",
  "仪","君","艺","莹","曼","悦","菲","梦","婉","柔",
  "安","宁","馨","怡","诗","琪","雅","琳","思","晓",
  "美","然","盈","若","竹","含","烟","荷","彤","萱",
  "碧","采","雁","真","巧","环","翠","婵","妃","沁",
  "淑","惠","菁","贞","苑","利","伊","凡","之","如",
  "珠","爱","璧","璧","蕊","芬","淼","宜","妍","勤",
  "叶","枫","灵","绿","夏","秋","冬","寒","冰","凝",
  "语","言","画","白","紫","蓝","青","幼","代","双",
  "山","水","璧","天","香","惜","问","优,"依","惜",
  "妙","寻","谷","向","映","新","书","智","妲","嫒",
  "可","姿","伊","璧","琼","璐","瑶","薇","蔷","瑰"
];
```

- [ ] **Step 3: 浏览器打开验证无 JS 报错**

打开浏览器控制台，确认 `REGION_DATA`、`SURNAMES`、`MALE_NAMES`、`FEMALE_NAMES` 均可访问。

---

### Task 3: JavaScript 核心逻辑

**Files:**
- Modify: `idcard-generator.html` — 在数据常量之后追加核心函数

**Interfaces:**
- Produces:
  - `function calcChecksum(id17: string): string` — 返回 1 位校验码字符（'0'-'9' 或 'X'）
  - `function generateSequenceCode(gender: 'male'|'female'): string` — 返回 3 位顺序码字符串
  - `function generateName(gender: 'male'|'female'): string` — 返回随机中文姓名
  - `function generateIdNumber(regionCode: string, birth: string, gender: 'male'|'female'): string` — 返回完整 18 位身份证号

- Consumes: `REGION_DATA`, `SURNAMES`, `MALE_NAMES`, `FEMALE_NAMES` (Task 2)

- [ ] **Step 1: 编写校验码计算函数**

```js
/* ========== 核心生成逻辑 ========== */

/**
 * 计算身份证第 18 位校验码（GB 11643 / ISO 7064:1983 MOD 11-2）
 * @param {string} id17 - 前 17 位身份证号
 * @returns {string} 校验码字符 '0'-'9' 或 'X'
 */
function calcChecksum(id17) {
  const weights = [7, 9, 10, 5, 8, 4, 2, 1, 6, 3, 7, 9, 10, 5, 8, 4, 2];
  const checkMap = ['1', '0', 'X', '9', '8', '7', '6', '5', '4', '3', '2'];
  let sum = 0;
  for (let i = 0; i < 17; i++) {
    sum += parseInt(id17[i], 10) * weights[i];
  }
  return checkMap[sum % 11];
}
```

- [ ] **Step 2: 编写顺序码生成函数**

```js
/**
 * 生成 3 位顺序码，第 3 位奇偶匹配性别
 * @param {'male'|'female'} gender
 * @returns {string} 3 位数字字符串
 */
function generateSequenceCode(gender) {
  // 随机生成前 2 位 (00-99)
  const prefix = String(Math.floor(Math.random() * 100)).padStart(2, '0');
  // 第 3 位：男性奇数(1,3,5,7,9)，女性偶数(0,2,4,6,8)
  const oddDigits = [1, 3, 5, 7, 9];
  const evenDigits = [0, 2, 4, 6, 8];
  const pool = gender === 'male' ? oddDigits : evenDigits;
  const last = pool[Math.floor(Math.random() * pool.length)];
  return prefix + last;
}
```

- [ ] **Step 3: 编写姓名生成函数**

```js
/**
 * 根据性别随机生成中文姓名
 * @param {'male'|'female'} gender
 * @returns {string}
 */
function generateName(gender) {
  const surname = SURNAMES[Math.floor(Math.random() * SURNAMES.length)];
  const namePool = gender === 'male' ? MALE_NAMES : FEMALE_NAMES;
  // 随机 1~2 个名字字
  const count = Math.random() < 0.5 ? 1 : 2;
  let givenName = '';
  const used = new Set();
  for (let i = 0; i < count; i++) {
    let ch;
    // 避免重复选到同一个字
    do { ch = namePool[Math.floor(Math.random() * namePool.length)]; } while (used.has(ch));
    used.add(ch);
    givenName += ch;
  }
  return surname + givenName;
}
```

- [ ] **Step 4: 编写完整身份证号生成函数**

```js
/**
 * 生成完整 18 位身份证号
 * @param {string} regionCode - 6 位地址码
 * @param {string} birth - 8 位出生日期码 YYYYMMDD
 * @param {'male'|'female'} gender
 * @returns {string} 18 位身份证号
 */
function generateIdNumber(regionCode, birth, gender) {
  const id17 = regionCode + birth + generateSequenceCode(gender);
  return id17 + calcChecksum(id17);
}
```

- [ ] **Step 5: 控制台手动验证**

打开浏览器控制台，执行：

```js
// 测试校验码：已知合法号码的校验码应匹配
calcChecksum('11010119900101001')  // 应返回正确的校验码字符

// 测试顺序码性别匹配
const seq = generateSequenceCode('male');
console.log('male seq:', seq, 'last digit odd:', parseInt(seq[2]) % 2 === 1);
const seq2 = generateSequenceCode('female');
console.log('female seq:', seq2, 'last digit even:', parseInt(seq2[2]) % 2 === 0);

// 测试完整生成
generateIdNumber('110101', '19900101', 'male');
// 应返回 18 位，末位校验正确
```

---

### Task 4: JavaScript UI 交互

**Files:**
- Modify: `idcard-generator.html` — 在核心逻辑之后追加 UI 交互代码

**Interfaces:**
- Consumes: `REGION_DATA` (Task 2), `generateIdNumber()`, `generateName()`, `calcChecksum()` (Task 3)
- Produces: UI 事件绑定 — 地区联动、日期联动、生成按钮、复制按钮

- [ ] **Step 1: 初始化地区下拉框**

```js
/* ========== UI 交互 ========== */
const $ = (id) => document.getElementById(id);

const elProvince = $('province');
const elCity = $('city');
const elDistrict = $('district');
const elYear = $('year');
const elMonth = $('month');
const elDay = $('day');
const elGender = $('gender');
const elBtnGenerate = $('btn-generate');
const elHintAddress = $('hint-address');
const elResultContainer = $('result-container');

/* --- 地区三级联动 --- */

// 填充省列表
function initProvinces() {
  elProvince.innerHTML = '<option value="">请选择省</option>';
  for (const [name, data] of Object.entries(REGION_DATA)) {
    const opt = document.createElement('option');
    opt.value = name;
    opt.textContent = name;
    elProvince.appendChild(opt);
  }
}

// 省变更 → 刷新市列表
function onProvinceChange() {
  const provinceName = elProvince.value;
  elCity.innerHTML = '<option value="">请选择市</option>';
  elDistrict.innerHTML = '<option value="">请选择区</option>';
  elDistrict.disabled = true;
  elBtnGenerate.disabled = true;
  elHintAddress.classList.add('visible');

  if (!provinceName) {
    elCity.disabled = true;
    return;
  }
  elCity.disabled = false;
  const cities = REGION_DATA[provinceName].cities;
  for (const [name, data] of Object.entries(cities)) {
    const opt = document.createElement('option');
    opt.value = name;
    opt.textContent = name;
    elCity.appendChild(opt);
  }
}

// 市变更 → 刷新区列表
function onCityChange() {
  const provinceName = elProvince.value;
  const cityName = elCity.value;
  elDistrict.innerHTML = '<option value="">请选择区</option>';
  elBtnGenerate.disabled = true;
  elHintAddress.classList.add('visible');

  if (!cityName) {
    elDistrict.disabled = true;
    return;
  }
  elDistrict.disabled = false;
  const districts = REGION_DATA[provinceName].cities[cityName].districts;
  for (const [name, code] of Object.entries(districts)) {
    const opt = document.createElement('option');
    opt.value = code; // 区级存的是地址码
    opt.textContent = name;
    elDistrict.appendChild(opt);
  }
}

// 区变更 → 地址选择完成
function onDistrictChange() {
  if (elDistrict.value) {
    elBtnGenerate.disabled = false;
    elHintAddress.classList.remove('visible');
  } else {
    elBtnGenerate.disabled = true;
    elHintAddress.classList.add('visible');
  }
}

elProvince.addEventListener('change', onProvinceChange);
elCity.addEventListener('change', onCityChange);
elDistrict.addEventListener('change', onDistrictChange);
```

- [ ] **Step 2: 初始化日期选择器**

```js
/* --- 日期选择器 --- */

const now = new Date();
const currentYear = now.getFullYear();

// 判断闰年
function isLeapYear(year) {
  return (year % 4 === 0 && year % 100 !== 0) || year % 400 === 0;
}

// 获取某月天数
function getDaysInMonth(year, month) {
  const days = [31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31];
  if (month === 2 && isLeapYear(year)) return 29;
  return days[month - 1];
}

// 填充年份（1949 ~ 当前年）
function initYears() {
  for (let y = currentYear; y >= 1949; y--) {
    const opt = document.createElement('option');
    opt.value = y;
    opt.textContent = y;
    elYear.appendChild(opt);
  }
}

// 填充月份
function initMonths() {
  for (let m = 1; m <= 12; m++) {
    const opt = document.createElement('option');
    opt.value = m;
    opt.textContent = String(m).padStart(2, '0');
    elMonth.appendChild(opt);
  }
}

// 刷新日列表（根据当前选中的年月）
function refreshDays() {
  const year = parseInt(elYear.value, 10);
  const month = parseInt(elMonth.value, 10);
  const selectedDay = elDay.value;
  const maxDay = getDaysInMonth(year, month);

  elDay.innerHTML = '';
  for (let d = 1; d <= maxDay; d++) {
    const opt = document.createElement('option');
    opt.value = d;
    opt.textContent = String(d).padStart(2, '0');
    elDay.appendChild(opt);
  }
  // 恢复之前选中的日（如果仍有效）
  if (selectedDay && parseInt(selectedDay, 10) <= maxDay) {
    elDay.value = selectedDay;
  }
}

elYear.addEventListener('change', refreshDays);
elMonth.addEventListener('change', refreshDays);
```

- [ ] **Step 3: 编写结果渲染函数**

```js
/* --- 结果渲染 --- */

/**
 * 根据选项生成完整结果并渲染到页面
 */
function renderResult() {
  const gender = elGender.value;
  const year = elYear.value;
  const month = String(elMonth.value).padStart(2, '0');
  const day = String(elDay.value).padStart(2, '0');
  const birth = year + month + day;
  const regionCode = elDistrict.value; // 区级 value 就是地址码

  const provinceName = elProvince.value;
  const cityName = elCity.value;
  // 获取区名（从 value 反查）
  const districts = REGION_DATA[provinceName].cities[cityName].districts;
  const districtName = Object.keys(districts).find(k => districts[k] === regionCode);
  const fullAddress = provinceName + cityName + districtName;

  const idNumber = generateIdNumber(regionCode, birth, gender);
  const name = generateName(gender);
  const genderText = gender === 'male' ? '男' : '女';

  elResultContainer.innerHTML = `
    <div class="result-item">
      <span class="result-label">姓名</span>
      <span class="result-value">${name}</span>
    </div>
    <div class="result-item">
      <span class="result-label">性别</span>
      <span class="result-value">${genderText}</span>
    </div>
    <div class="result-item">
      <span class="result-label">民族</span>
      <span class="result-value">汉族</span>
    </div>
    <div class="result-item">
      <span class="result-label">出生日期</span>
      <span class="result-value">${year}年${month}月${day}日</span>
    </div>
    <div class="result-item">
      <span class="result-label">住址</span>
      <span class="result-value">${fullAddress}</span>
    </div>
    <div class="result-item">
      <span class="result-label">身份证号</span>
      <span class="result-value id-number">${idNumber}</span>
      <button class="btn-copy" onclick="copyIdNumber(this, '${idNumber}')">复制</button>
    </div>
  `;
}

elBtnGenerate.addEventListener('click', renderResult);
```

- [ ] **Step 4: 编写复制功能函数**

```js
/* --- 复制功能 --- */

function copyIdNumber(btn, idNumber) {
  if (navigator.clipboard && navigator.clipboard.writeText) {
    navigator.clipboard.writeText(idNumber).then(() => {
      showCopied(btn);
    }).catch(() => {
      fallbackCopy(btn, idNumber);
    });
  } else {
    fallbackCopy(btn, idNumber);
  }
}

function showCopied(btn) {
  btn.textContent = '已复制';
  btn.classList.add('copied');
  setTimeout(() => {
    btn.textContent = '复制';
    btn.classList.remove('copied');
  }, 1500);
}

// 降级方案：选中文本提示手动复制
function fallbackCopy(btn, idNumber) {
  const range = document.createRange();
  const textNode = btn.parentElement.querySelector('.id-number');
  range.selectNodeContents(textNode);
  const sel = window.getSelection();
  sel.removeAllRanges();
  sel.addRange(range);
  showCopied(btn);
  // 仍给 1.5 秒选中状态供手动 Ctrl+C
  setTimeout(() => sel.removeAllRanges(), 1500);
}
```

- [ ] **Step 5: 关闭 script 标签，加入页面初始化调用**

```js
/* --- 初始化 --- */
initProvinces();
initYears();
initMonths();
refreshDays();
```

然后补上 `</script>` 标签。

---

### Task 5: 默认值填充 + 首页自动生成

**Files:**
- Modify: `idcard-generator.html` — 在初始化代码后追加默认值设置和首次自动生成

**Interfaces:**
- Consumes: 所有 Task 2-4 中定义的函数和 DOM 元素引用

- [ ] **Step 1: 在初始化代码之后追加默认值填充和首次生成**

```js
/* --- 默认值填充 & 首次自动生成 --- */

function setDefaults() {
  // 默认性别：男
  elGender.value = 'male';

  // 默认出生日期：1990-01-01
  elYear.value = '1990';
  elMonth.value = '1';
  refreshDays();
  elDay.value = '1';

  // 默认地区：北京市-市辖区-东城区
  elProvince.value = '北京市';
  onProvinceChange();
  elCity.value = '市辖区';
  onCityChange();
  elDistrict.value = '110101';
  onDistrictChange();
}

setDefaults();
renderResult();
```

- [ ] **Step 2: 浏览器打开验证最终效果**

打开 `idcard-generator.html`：
- 首屏应显示默认值（男/1990-01-01/北京市市辖区东城区）并已生成一张身份证结果
- 切换地区：选省后市区联动正常，地址未完整时生成按钮禁用
- 切换日期：选 2 月时，日最多 28；选 2000 年 2 月时可选到 29 日
- 切换性别后点击生成，姓名性别匹配，身份证号第 17 位奇偶匹配
- 点击复制按钮，"复制"变为"已复制"，1.5 秒后恢复
