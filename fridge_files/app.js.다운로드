/* ============================================================ */
/*  FRS — Food Rescue Squad                                     */
/*  공통 비즈니스 로직 (localStorage 기반)                      */
/* ============================================================ */

/**
 * ===== 백엔드 아키텍처 (Express + PostgreSQL) =====
 *
 * [API 엔드포인트]
 *  POST /api/auth/register    - 회원가입 (body: {username,password,name})
 *  POST /api/auth/login       - 로그인 (body: {username,password}) → JWT 발급
 *  GET  /api/user/:id         - 유저 정보 조회 (header: Authorization)
 *  PATCH /api/user/:id        - 프로필 수정 (health/medications/allergies)
 *  POST /api/food             - 식재료 등록 (body: [FoodItem[]]) → +100 EP
 *  GET  /api/food/:userId     - 보관함 조회
 *  PATCH /api/food/:id        - 식재료 상태 변경 (consumed: true/false)
 *  DELETE /api/food/:id       - 식재료 삭제
 *  POST /api/receipt/scan     - 영수증 분석 (multipart: image) → AI Vision OCR
 *  GET  /api/stats/:userId    - 통계 조회 (savedItems, savedMoney, reducedCO2)
 *  POST /api/points/add       - 포인트 적립 (+100 per receipt scan, max 10000)
 *  POST /api/points/deduct    - 포인트 차감 (-50 per expired item)
 *
 * [DB 스키마]
 *  CREATE TABLE users (
 *    id SERIAL PRIMARY KEY,
 *    username VARCHAR(50) UNIQUE NOT NULL,
 *    password_hash VARCHAR(255) NOT NULL,
 *    name VARCHAR(50) NOT NULL,
 *    health TEXT DEFAULT '',
 *    medications TEXT DEFAULT '',
 *    allergies TEXT DEFAULT '',
 *    points INT DEFAULT 0 CHECK (points >= 0 AND points <= 10000),
 *    total_saved_items INT DEFAULT 0,
 *    total_saved_money INT DEFAULT 0,
 *    total_reduced_co2 DECIMAL(10,2) DEFAULT 0,
 *    created_at TIMESTAMP DEFAULT NOW()
 *  );
 *
 *  CREATE TABLE food_items (
 *    id SERIAL PRIMARY KEY,
 *    user_id INT REFERENCES users(id) ON DELETE CASCADE,
 *    name VARCHAR(100) NOT NULL,
 *    category VARCHAR(50),
 *    purchase_date DATE NOT NULL,
 *    expiry_date DATE NOT NULL,
 *    consumed BOOLEAN DEFAULT FALSE,
 *    created_at TIMESTAMP DEFAULT NOW()
 *  );
 *
 * [포인트/등급 로직 (서버 측)]
 *  - 영수증 등록 시 +100 EP (단, 실제 식품이 포함된 경우에만)
 *  - 만료된 식품 발견 시 품목당 -50 EP (1일 1회만 적용)
 *  - 최대 포인트 10,000 EP
 */

/* ===== localStorage 키 ===== */
const KEY_FOOD      = 'frs_food_items';
const KEY_USERS     = 'frs_users';
const KEY_SESSION   = 'frs_session';
const KEY_POINTS    = 'frs_points';
const KEY_SELECTED  = 'frs_selected_ingredients';
const KEY_PENALTY   = 'frs_penalty_applied'; // 페널티 적용된 식품 ID 목록
const KEY_STATS     = 'frs_stats';           // { savedItems, savedMoney, reducedCO2 }

/* ===== DOM ===== */
const $ = (s) => document.querySelector(s);
const $$ = (s) => document.querySelectorAll(s);

/* ================================================================
   Earth Point 시스템 (최대 10,000 EP)
   ================================================================ */
function getPoints() {
  return Math.min(parseInt(localStorage.getItem(KEY_POINTS) || '0'), 10000);
}
function setPoints(val) {
  localStorage.setItem(KEY_POINTS, String(Math.min(Math.max(val, 0), 10000)));
}
function addPoints(amount) {
  const pts = getPoints() + amount;
  setPoints(pts);
  return getPoints();
}
function deductPoints(amount) {
  const pts = getPoints() - amount;
  setPoints(pts);
  return getPoints();
}

/* ================================================================
   등급 체계
   ================================================================ */
const TIERS = [
  { min: 9000, max: 10000, name: '역대급 레전드 종결 초토화 지구 지킴이', emoji: '👑', color: '#e8a838', benefit: '명예의 전당 닉네임 노출' },
  { min: 6000, max: 8999,  name: '탄소 제로 마스터',emoji: '⚡', color: '#38b8d8', benefit: '환경 단체 기부 인증서 발급' },
  { min: 3000, max: 5999,  name: '지구 수호자',     emoji: '🌍', color: '#3a9',    benefit: '한정판 프로필 스킨 해제' },
  { min: 1000, max: 2999,  name: '베테랑 요원',     emoji: '🥦', color: '#6a6',    benefit: '그린 커뮤니티 작성 권한 오픈' },
  { min: 0,    max: 999,   name: '새싹 구조대원',   emoji: '🌱', color: '#999',    benefit: '기본 프로필 테마' },
];
function getUserTier(pts) {
  for (const t of TIERS) { if (pts >= t.min && pts <= t.max) return t; }
  return TIERS[TIERS.length - 1];
}

/* ================================================================
   지구 구출 통계
   ================================================================ */
function getStats() {
  try { return JSON.parse(localStorage.getItem(KEY_STATS)) || { savedItems: 0, savedMoney: 0, reducedCO2: 0 }; } catch(e) { return { savedItems: 0, savedMoney: 0, reducedCO2: 0 }; }
}
function saveStats(stats) { localStorage.setItem(KEY_STATS, JSON.stringify(stats)); }
function addStats(itemCount) {
  const s = getStats();
  s.savedItems += itemCount;
  s.savedMoney += itemCount * 3000;    // 품목당 평균 3,000원 절약
  s.reducedCO2 += itemCount * 2.3;     // 품목당 평균 2.3 kg CO2 감소
  saveStats(s);
}

/* ================================================================
   식재료 데이터
   ================================================================ */
function getFoodItems() {
  try { return JSON.parse(localStorage.getItem(KEY_FOOD) || '[]'); } catch(e) { return []; }
}
function saveFoodItems(items) { localStorage.setItem(KEY_FOOD, JSON.stringify(items)); }
function addFoodItems(newItems) {
  const items = getFoodItems();
  const now = new Date();
  const ts = `${now.getFullYear()}-${String(now.getMonth()+1).padStart(2,'0')}-${String(now.getDate()).padStart(2,'0')}`;
  newItems.forEach((item, i) => {
    items.push({ id: Date.now() + i, name: item.name, category: item.category, purchaseDate: item.purchaseDate || ts, expiryDate: item.expiryDate, icon: item.icon || getIcon(item.category), consumed: false });
  });
  saveFoodItems(items);
}
function getIcon(cat) {
  if (!cat) return '📦';
  if (cat.includes('육류')||cat.includes('생선')) return '🥩';
  if (cat.includes('우유')||cat.includes('유제품')) return '🥛';
  if (cat.includes('두부')||cat.includes('콩나물')) return '🧈';
  if (cat.includes('계란')) return '🥚';
  if (cat.includes('채소')||cat.includes('과일')) return '🥬';
  if (cat.includes('냉동')) return '🧊'; if (cat.includes('가공')) return '🥫';
  if (cat.includes('저장')) return '🧅'; if (cat.includes('곡류')||cat.includes('빵')) return '🍞';
  return '📦';
}

/* ================================================================
   페널티: 만료된 식품에 -50 EP (1일 1회)
   ================================================================ */
function getPenaltyApplied() {
  try { return JSON.parse(localStorage.getItem(KEY_PENALTY) || '[]'); } catch(e) { return []; }
}
function setPenaltyApplied(ids) { localStorage.setItem(KEY_PENALTY, JSON.stringify(ids)); }

function applyExpiryPenalties() {
  const today = new Date(); today.setHours(0,0,0,0);
  const items = getFoodItems();
  const expired = items.filter(f => {
    if (f.consumed) return false;
    return new Date(f.expiryDate) < today;
  });
  if (expired.length === 0) return { count: 0, penalty: 0 };

  const applied = getPenaltyApplied();
  const newExpired = expired.filter(f => !applied.includes(f.id));
  if (newExpired.length === 0) return { count: 0, penalty: 0 };

  const penalty = newExpired.length * 50;
  deductPoints(penalty);

  // 페널티 적용된 ID 기록
  const allApplied = [...new Set([...applied, ...newExpired.map(f => f.id)])];
  setPenaltyApplied(allApplied);

  return { count: newExpired.length, penalty };
}

/* ================================================================
   소비 완료 시 통계 반영
   ================================================================ */
function onConsumeItem(item) {
  addStats(1); // 소비 성공 → 통계 +1
}

/* ================================================================
   사용자 인증
   ================================================================ */
function getUsers() { try { return JSON.parse(localStorage.getItem(KEY_USERS) || '[]'); } catch(e) { return []; } }
function saveUsers(u) { localStorage.setItem(KEY_USERS, JSON.stringify(u)); }
function getSession() { try { return JSON.parse(localStorage.getItem(KEY_SESSION)); } catch(e) { return null; } }
function saveSession(u) { localStorage.setItem(KEY_SESSION, JSON.stringify(u)); }

/* ================================================================
   선택된 레시피 재료
   ================================================================ */
function setSelectedIngredients(ids) { localStorage.setItem(KEY_SELECTED, JSON.stringify(ids)); }
function getSelectedIngredients() { try { return JSON.parse(localStorage.getItem(KEY_SELECTED) || '[]'); } catch(e) { return []; } }

/* ================================================================
   공통 네비게이션
   ================================================================ */
function renderNavbar(currentPage) {
  const nav = document.getElementById('navbar');
  if (!nav) return;
  const user = getSession();
  const pages = [
    { name:'대시보드', href:'index.html', icon:'📊' },
    { name:'영수증 등록', href:'scan.html', icon:'🧾' },
    { name:'보관함', href:'fridge.html', icon:'📦' },
  ];
  let links = pages.map(p => `<a href="${p.href}" class="${currentPage === p.href.split('.')[0] ? 'active' : ''}">${p.icon} ${p.name}</a>`).join('');

  let ua = '';
  if (user) {
    ua = `<span class="user-name">👤 ${esc(user.name||user.username)}</span>
      <button class="user-btn" onclick="event.preventDefault();showProfileModal()">⚙️</button>
      <button class="user-btn" onclick="event.preventDefault();doLogout()">로그아웃</button>`;
  } else {
    ua = `<button class="user-btn" onclick="event.preventDefault();showLoginModal()">로그인</button>
      <button class="user-btn" onclick="event.preventDefault();showRegisterModal()">회원가입</button>`;
  }

  nav.innerHTML = `<a href="index.html" class="logo">🛒 FRS</a><div class="nav-links">${links}</div>${ua}`;
}

/* ================================================================
   토스트
   ================================================================ */
let _toastTm;
function showToast(msg) {
  const el = document.getElementById('toast');
  if (!el) return;
  clearTimeout(_toastTm);
  el.querySelector('span').textContent = msg;
  el.classList.add('show');
  _toastTm = setTimeout(() => el.classList.remove('show'), 2400);
}

/* ================================================================
   모달
   ================================================================ */
function openModal(id) { const el = document.getElementById(id); if (el) el.classList.add('on'); }
function closeModal(id) { const el = document.getElementById(id); if (el) el.classList.remove('on'); }
function closeRecipeModal() { const m = document.getElementById('recipeModal'); if (m) m.classList.remove('on'); }

/* ================================================================
   인증 액션
   ================================================================ */
function showLoginModal() { closeModal('registerModal'); closeModal('profileModal'); openModal('loginModal'); setTimeout(()=>{const e=$('#loginId');if(e)e.focus();},100); }
function showRegisterModal() { closeModal('loginModal'); closeModal('profileModal'); openModal('registerModal'); setTimeout(()=>{const e=$('#regId');if(e)e.focus();},100); }
function showProfileModal() {
  const u = getSession(); if (!u) return showToast('먼저 로그인해 주세요.');
  const map = { profileHealth:'health', profileMeds:'medications', profileAllergy:'allergies' };
  Object.entries(map).forEach(([id, key]) => { const e = $('#'+id); if(e) e.value = u[key] || ''; });
  closeModal('loginModal'); closeModal('registerModal'); openModal('profileModal');
}
function doLogin() {
  const id=($('#loginId')?.value||'').trim(), pw=$('#loginPw')?.value||'';
  if(!id||!pw) return showToast('아이디/비밀번호를 입력하세요.');
  const u = getUsers().find(x => x.username===id && x.password===pw);
  if(!u) return showToast('아이디 또는 비밀번호가 틀렸습니다.');
  saveSession({id:u.id,username:u.username,name:u.name,health:u.health,medications:u.medications,allergies:u.allergies});
  closeModal('loginModal'); renderNavbar(window._currentPage||''); showToast(`${u.name||u.username}님 환영합니다!`);
}
function doRegister() {
  const id=($('#regId')?.value||'').trim(), pw=$('#regPw')?.value||'', name=($('#regName')?.value||'').trim();
  if(!id||!pw||!name) return showToast('모든 항목을 입력하세요.');
  const users=getUsers(); if(users.find(x=>x.username===id)) return showToast('이미 사용 중인 아이디입니다.');
  const nu={id:Date.now(),username:id,password:pw,name,health:'',medications:'',allergies:''};
  users.push(nu); saveUsers(users);
  saveSession({id:nu.id,username:nu.username,name:nu.name,health:'',medications:'',allergies:''});
  closeModal('registerModal'); renderNavbar(window._currentPage||''); showToast(`${name}님 가입을 환영합니다!`);
}
function doLogout() { localStorage.removeItem(KEY_SESSION); renderNavbar(window._currentPage||''); showToast('로그아웃 되었습니다.'); }
function saveProfile() {
  const u = getSession(); if(!u) return;
  u.health=($('#profileHealth')?.value||'').trim(); u.medications=($('#profileMeds')?.value||'').trim(); u.allergies=($('#profileAllergy')?.value||'').trim();
  saveSession(u); const users=getUsers(); const i=users.findIndex(x=>x.id===u.id);
  if(i>=0){users[i].health=u.health;users[i].medications=u.medications;users[i].allergies=u.allergies;saveUsers(users);}
  closeModal('profileModal'); showToast('프로필이 저장되었습니다!');
}

/* ================================================================
   영수증 분석 (Mockup)
   ================================================================ */
function analyzeReceiptMock(text) {
  const today = new Date(); today.setHours(0,0,0,0);
  const fmt = d => `${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}-${String(d.getDate()).padStart(2,'0')}`;
  const ts = fmt(today);
  const db = [
    [['우유','서울우유','매일우유','요구르트','요거트','요플레','치즈','버터','생크림'],'우유/유제품',14,'🥛'],
    [['삼겹살','목살','소고기','돼지고기','닭고기','닭다리','닭가슴살','생선','고등어','갈치','연어','참치','참치회','오징어','새우','스테이크','불고기','차돌박이'],'육류/생선',5,'🥩'],
    [['두부','콩나물','숙주'],'두부/콩나물',15,'🧈'],
    [['계란','달걀'],'계란',30,'🥚'],
    [['시금치','상추','깻잎','배추','양상추','로메인','케일','부추','미나리','쑥갓','버섯','느타리','팽이버섯','새송이','표고버섯','딸기','바나나','블루베리','라즈베리','체리','포도','복숭아','자두'],'잎채소/과일',7,'🥬'],
    [['사과','청송사과','배','감귤','귤','오렌지','자몽','레몬','키위','석류','망고','파인애플'],'과일(저장성)',21,'🍎'],
    [['만두','냉동만두','냉동','피자','치킨','핫도그','냉동피자','감자튀김','아이스크림'],'냉동식품',180,'🧊'],
    [['참치캔','통조림','햄','스팸','소시지','베이컨','라면','즉석밥','짜장','카레','레토르트','시리얼','과자','비스킷','초콜릿','사탕'],'가공식품',180,'🥫'],
    [['양파','마늘','감자','고구마','호박','단호박','당근','무','생강','옥수수'],'저장채소',60,'🧅'],
    [['쌀','잡곡','현미','밀가루','빵','식빵','베이글','파스타','국수','소면'],'곡류/빵',30,'🍞'],
  ];
  const lines = text.split(/[\n,]+/).map(l=>l.trim()).filter(l=>l.length>0 && !/^(합계|카드|현금|할인|적립|포인트|총|===|---)/.test(l) && !/^[\d,]+원?$/.test(l));
  const found=[], used=new Set();
  for(const line of lines){ for(const [kws,cat,days,icon] of db){ for(const kw of kws){ if(line.includes(kw)&&!used.has(kw)){ used.add(kw); const exp=new Date(today); exp.setDate(exp.getDate()+days); found.push({name:kw,category:cat,purchaseDate:ts,expiryDate:fmt(exp),icon}); break; } } } }
  const uniq=[]; const seen=new Set(); for(const it of found){ if(!seen.has(it.name)){seen.add(it.name);uniq.push(it);} }
  if(uniq.length===0) return defaultMock(ts,today,fmt);
  return uniq;
}
function defaultMock(ts,t,fmt){ const add=d=>{const dt=new Date(t);dt.setDate(dt.getDate()+d);return fmt(dt);}; return [{name:'우유',category:'우유/유제품',purchaseDate:ts,expiryDate:add(14),icon:'🥛'},{name:'삼겹살',category:'육류/생선',purchaseDate:ts,expiryDate:add(5),icon:'🥩'},{name:'사과',category:'과일(저장성)',purchaseDate:ts,expiryDate:add(21),icon:'🍎'},{name:'두부',category:'두부/콩나물',purchaseDate:ts,expiryDate:add(15),icon:'🧈'},{name:'계란',category:'계란',purchaseDate:ts,expiryDate:add(30),icon:'🥚'},{name:'시금치',category:'잎채소/과일',purchaseDate:ts,expiryDate:add(7),icon:'🥬'},{name:'냉동만두',category:'냉동식품',purchaseDate:ts,expiryDate:add(180),icon:'🧊'}]; }
function getFakeReceipt() { const items=['서울우유 1L\t2,980','삼겹살 500g\t12,000','청송사과 3개\t5,000','두부 1모\t1,500','계란 10구\t3,500','시금치 1단\t2,000','냉동만두 1kg\t8,000','바나나 1송이\t2,500','양파 3개\t1,800','참치캔 3개\t4,500']; const shop=['든든마트','초록마트','행복슈퍼'][Math.floor(Math.random()*3)]; const now=new Date(); const d=`${now.getFullYear()}-${String(now.getMonth()+1).padStart(2,'0')}-${String(now.getDate()).padStart(2,'0')}`; return `=== ${shop} ===\n날짜: ${d}\n\n${items.join('\n')}\n\n합계: 43,780원\n카드결제`; }

/* ================================================================
   레시피 생성
   ================================================================ */
function generateRecipeGroups(ingredients) {
  const ms=['삼겹살','목살','소고기','돼지고기','닭고기','닭다리','닭가슴살','불고기','차돌박이','스테이크'];
  const fs=['생선','고등어','갈치','연어','참치','참치회','오징어','새우'];
  const ls=['시금치','상추','깻잎','배추','양상추','로메인','케일','부추','미나리','쑥갓'];
  const rs=['양파','당근','무','감자','고구마','호박','단호박','마늘','생강'];
  const ss=['버섯','느타리','팽이버섯','새송이','표고버섯'];
  const ts_=['두부','콩나물','숙주'];
  const frs=['사과','청송사과','바나나','딸기','블루베리','오렌지','귤','감귤','포도','복숭아','배','키위','석류','망고','파인애플','레몬','자몽','체리','라즈베리','자두','살구'];
  const fzs=['만두','냉동만두','냉동피자','치킨','핫도그','감자튀김','아이스크림'];
  const cs=['참치캔','통조림','햄','스팸','소시지','베이컨','라면','즉석밥','짜장','카레','레토르트','시리얼','과자','비스킷','초콜릿','사탕'];
  const gs=['쌀','잡곡','현미','밀가루','빵','식빵','베이글','파스타','국수','소면'];
  const es=['계란','달걀']; const mks=['우유','서울우유','매일우유','요구르트','요거트'];
  const meats=ingredients.filter(n=>ms.includes(n)), fishes=ingredients.filter(n=>fs.includes(n));
  const leafs=ingredients.filter(n=>ls.includes(n)), roots=ingredients.filter(n=>rs.includes(n));
  const shrooms=ingredients.filter(n=>ss.includes(n)), tofus=ingredients.filter(n=>ts_.includes(n));
  const fruits=ingredients.filter(n=>frs.includes(n)), frozens=ingredients.filter(n=>fzs.includes(n));
  const canneds=ingredients.filter(n=>cs.includes(n)), grains=ingredients.filter(n=>gs.includes(n));
  const eggs=ingredients.filter(n=>es.includes(n)), milks=ingredients.filter(n=>mks.includes(n));
  const allVeg=[...leafs,...roots,...shrooms];
  const used=new Set(), groups=[];
  const use=ns=>ns.forEach(n=>used.add(n));
  if(meats.length>0&&allVeg.length>=1){const grp=[...meats,...allVeg.slice(0,3)];if(tofus.length>0)grp.push(tofus[0]);if(eggs.length>0&&grp.length<5)grp.push(eggs[0]);use(grp);groups.push({items:[...new Set(grp)],type:'meat_main'});}
  if(fishes.length>0&&!used.has(fishes[0])){const rv=allVeg.filter(v=>!used.has(v));const grp=[fishes[0]];if(rv.length>=1)grp.push(rv[0]);if(rv.length>=2)grp.push(rv[1]);use(grp);groups.push({items:[...new Set(grp)],type:'seafood'});}
  const uf=frozens.filter(f=>!used.has(f));if(uf.length>0){const rv=allVeg.filter(v=>!used.has(v));const grp=[uf[0]];if(rv.length>=1)grp.push(rv[0]);if(rv.length>=2)grp.push(rv[1]);if(tofus.filter(t=>!used.has(t)).length>0)grp.push(tofus.filter(t=>!used.has(t))[0]);use(grp);groups.push({items:[...new Set(grp)],type:'frozen'});}
  const uc=canneds.filter(c=>!used.has(c));if(uc.length>0){const rv=allVeg.filter(v=>!used.has(v));const grp=[uc[0]];if(rv.length>=1)grp.push(rv[0]);if(rv.length>=2)grp.push(rv[1]);if(tofus.filter(t=>!used.has(t)).length>0)grp.push(tofus.filter(t=>!used.has(t))[0]);use(grp);groups.push({items:[...new Set(grp)],type:'canned'});}
  const ufr=fruits.filter(f=>!used.has(f)), um=milks.filter(m=>!used.has(m));if(ufr.length>0&&um.length>0){const grp=[ufr[0],um[0]];if(ufr.length>=2)grp.push(ufr[1]);use(grp);groups.push({items:[...new Set(grp)],type:'dessert'});}else if(ufr.length>0&&groups.length===0){const grp=ufr.slice(0,3);use(grp);groups.push({items:[...new Set(grp)],type:'fruit_only'});}
  const ug=grains.filter(g=>!used.has(g)), ue=eggs.filter(e=>!used.has(e)), rv2=allVeg.filter(v=>!used.has(v));if(ug.length>0&&rv2.length>=1){const grp=[ug[0],...rv2.slice(0,3)];if(ue.length>0)grp.push(ue[0]);use(grp);groups.push({items:[...new Set(grp)],type:'grain'});}
  const re2=eggs.filter(e=>!used.has(e)), rv3=allVeg.filter(v=>!used.has(v));if(re2.length>0&&rv3.length>=1){const grp=[re2[0],...rv3.slice(0,2)];use(grp);groups.push({items:[...new Set(grp)],type:'egg'});}
  const rt=tofus.filter(t=>!used.has(t)), rv4=allVeg.filter(v=>!used.has(v));if(rt.length>0&&rv4.length>=1){const grp=[rt[0],...rv4.slice(0,2)];use(grp);groups.push({items:[...new Set(grp)],type:'tofu_main'});}
  const rem=ingredients.filter(n=>!used.has(n));if(rem.length>0){if(groups.length===0){use(rem);groups.push({items:rem,type:'all'});}else{if(rem.some(n=>frs.includes(n))){const fr=rem.filter(n=>frs.includes(n));use(fr);groups.push({items:fr,type:'fruit_only'});}const sr2=rem.filter(n=>!used.has(n));if(sr2.length>0){use(sr2);groups.push({items:sr2,type:'misc'});}}}
  return groups.map(g=>generateSingleRecipe(g));
}
function generateSingleRecipe(group) {
  const items=group.items.filter(Boolean), set=new Set(items), names=items.join(', ');
  const is=l=>l.some(k=>set.has(k)), pick=l=>l.find(k=>set.has(k))||'';
  const ms=['삼겹살','목살','소고기','돼지고기','닭고기','닭다리','닭가슴살','불고기','차돌박이','스테이크'];
  const fs=['생선','고등어','갈치','연어','참치','참치회','오징어','새우'];
  const ls=['시금치','상추','깻잎','배추','양상추','로메인','케일','부추','미나리','쑥갓'];
  const rs=['양파','당근','무','감자','고구마','호박','단호박'];
  const ss=['버섯','느타리','팽이버섯','새송이','표고버섯'];
  const ts_=['두부','콩나물','숙주'];
  const frs=['사과','청송사과','바나나','딸기','블루베리','오렌지','귤','감귤','포도','복숭아','배','키위','망고','파인애플','레몬','자몽'];
  const fzs=['만두','냉동만두','냉동피자','치킨','핫도그','감자튀김','아이스크림'];
  const cs=['참치캔','통조림','햄','스팸','소시지','베이컨','라면','즉석밥','짜장','카레','레토르트','시리얼','과자','비스킷','초콜릿','사탕'];
  const gs=['쌀','잡곡','현미','밀가루','빵','식빵','베이글','파스타','국수','소면'];
  const hM=is(ms),hF=is(fs),hL=is(ls),hR=is(rs),hS=is(ss),hT=is(ts_);
  const hE=set.has('계란')||set.has('달걀'), hMi=set.has('우유')||set.has('서울우유')||set.has('매일우유');
  const hFr=is(frs),hFz=is(fzs),hC=is(cs),hG=is(gs);
  const vC=(hL?1:0)+(hR?1:0)+(hS?1:0);
  const detail=items.map(name=>({name,usage:makeUsage(name)}));
  let ttl,stp,time,serv,diff,tip;
  if(hM&&hT&&vC>=1){const m=pick(ms);ttl=`${m} 두부 모둠채소 전골`;time='25분';serv='2인분';diff='★★☆';stp=`1. ${m}는 핏물을 닦고 한입 크기로 썰어 소금·후추에 5분 밑간합니다.\n2. 두부는 1.5cm 두께로 썰고, 모든 채소는 먹기 좋게 손질하세요.\n3. 전골 냄비에 참기름을 두르고 ${m}를 중불에서 노릇하게 볶아요.\n4. 채소를 모두 넣고 2분 볶은 뒤 물 400ml와 국간장 1스푼, 다진 마늘을 넣고 팔팔 끓여요.\n5. 중약불로 8분 끓인 후 두부를 넣고 3분만 더 — 완성!`;tip=`두부는 마지막에 넣어야 부드럽습니다. 남은 국물은 칼국수 육수로 활용하세요.`;}
  else if(hM&&vC>=1&&!hT){const m=pick(ms);if(hL){ttl=`${m} 채소 강불 볶음`;time='20분';serv='2인분';diff='★★☆';stp=`1. ${m}는 얇게 저며 소금·후추·청주에 10분 재워둡니다.\n2. 잎채소는 5cm로, 뿌리채소는 채썰고 버섯은 찢어 준비하세요.\n3. 웍을 센 불로 달군 뒤 식용유 2스푼 — ${m}를 넣고 1분간 강볶.\n4. 뿌리채소→버섯 순으로 넣고 2분 볶은 후 굴소스1+간장1+설탕 약간.\n5. 잎채소를 마지막에 넣고 30초만 볶은 뒤 불을 끄고 참기름 — 완성!`;tip=`잎채소는 30초만 볶아야 아삭함이 유지됩니다.`;}else{ttl=`${m} 뿌리채소 구이 플레이트`;time='20분';serv='2인분';diff='★☆☆';stp=`1. ${m}는 상온에 20분 둔 뒤 소금·후추로 간하고 올리브오일을 발라요.\n2. 오븐을 200도로 예열합니다.\n3. 채소는 올리브오일·소금·후추를 뿌려 버무려 ${m}와 함께 팬에 올려요.\n4. ${m}는 한 면당 3~4분씩, 채소는 노릇하게 구워집니다.\n5. 접시에 채소를 깔고 고기를 올린 뒤 발사믹 글레이즈를 뿌려 마무리!`;tip=`고기 구운 팬에 물 2스푼으로 디글레이즈하면 즉석 소스 완성.`;}}
  else if(hF&&vC>=1){const f=pick(fs);ttl=`${f} 허브 채소 구이`;time='30분';serv='2인분';diff='★★☆';stp=`1. ${f}는 물기를 완전히 제거하고 소금·후추를 뿌려 10분 둡니다.\n2. 오븐을 200도로 예열하세요.\n3. 베이킹 시트에 채소를 깔고 올리브오일·소금·레몬즙을 뿌린 뒤 생선을 올려요.\n4. 레몬 슬라이스와 허브를 얹고 호일로 덮어 15분 구워요.\n5. 호일을 제거하고 5분 더 — 완성!`;tip=`생선구이의 핵심은 물기 제거입니다.`;}
  else if(hE&&vC>=1){ttl=`프렌치 오믈렛 with 제철 채소`;time='15분';serv='1~2인분';diff='★★★';stp=`1. 채소를 잘게 다지고, 계란 2알에 소금 한 꼬집+우유 1스푼을 넣어 풀어주세요.\n2. 버터를 녹인 팬에 중약불로 채소를 2분 볶아요.\n3. 계란물을 붓고 젓가락으로 저으며 가장자리부터 익혀요.\n4. 반숙 상태에서 불을 끄고 반으로 접어 접시에 담아요.\n5. 남은 채소를 올리고 파마산 치즈가루를 뿌려 마무리!`;tip=`계란은 센 불 금지! 중약불에서 천천히 익혀야 부드러워요.`;}
  else if(hFz&&vC>=1){const fz=pick(fzs);ttl=`${fz} 얼큰 전골`;time='25분';serv='2인분';diff='★☆☆';stp=`1. 깊은 냄비에 물 500ml+멸치 다시마 육수를 내고 끓여요.\n2. 채소를 전부 넣고 3분간 끓여 단맛을 우려냅니다.\n3. ${fz}을 해동 없이 그대로 넣고 5분간 중불에서 끓여요.\n4. 국간장 1스푼+다진 마늘 1작은술+고춧가루로 간을 맞추고 대파를 올려요.\n5. 2분 더 끓인 뒤 불을 끄고 후추 — 완성!`;tip=`냉동식품은 절대 해동하지 말고 얼린 채로 넣으세요.`;}
  else if(hC&&vC>=1){const c=pick(cs);ttl=`${c} 활용 매콤 찌개`;time='15분';serv='1~2인분';diff='★☆☆';stp=`1. 채소를 먹기 좋게 손질하세요.\n2. 냄비에 참기름을 두르고 채소를 중불에서 3분 볶아요.\n3. ${c}을 넣고 고춧가루+다진 마늘을 넣어 함께 볶아요.\n4. 물 400ml를 붓고 팔팔 끓어오르면 중약불로 7분 더 끓여요.\n5. 두부나 계란이 있으면 넣고 3분 더 — 완성!`;tip=`참치캔 기름은 버리지 말고 볶는 용도로 사용하면 감칠맛이 올라가요.`;}
  else if(hFr&&hMi){ttl=`트로피컬 프루트 스무디 볼`;time='10분';serv='1인분';diff='★☆☆';stp=`1. 과일을 씻어 한입 크기로 썰어 냉동실에 1시간 얼려요.\n2. 블렌더에 얼린 과일+우유200ml+꿀1스푼을 넣어요.\n3. 펄스 모드로 5~6회 갈아준 뒤 30초간 고속으로 곱게 갈아요.\n4. 볼에 담고 남은 과일을 토핑으로 올려 완성!`;tip=`과일을 미리 얼리면 얼음 없이도 크리미한 식감이 나요.`;}
  else if(hG&&vC>=1){const g=pick(gs);ttl=`불맛 가득 ${g} 채소 볶음밥`;time='15분';serv='2인분';diff='★★☆';stp=`1. 웍을 센 불로 예열하고 식용유 2스푼 — 계란이 있으면 스크램블해 덜어둬요.\n2. 같은 팬에 채소를 전부 넣고 강불에서 2분간 볶아요.\n3. ${g}을 넣고 주걱으로 눌러가며 2분 볶아줍니다.\n4. 간장1+굴소스1을 팬 가장자리에 둘러 향을 내고 계란을 다시 넣어요.\n5. 불을 끄고 참기름+통깨 — 완성!`;tip=`볶음밥은 센 불+빠른 손! 차가운 밥을 쓰면 더 고슬고슬해요.`;}
  else if(hT&&vC>=1){const t=pick(ts_);ttl=`${t} 채소 들기름 볶음`;time='15분';serv='2인분';diff='★☆☆';stp=`1. ${t}는 먹기 좋게 썰고, 채소도 손질하세요.\n2. 팬에 들기름을 두르고 뿌리채소→버섯 순으로 중불에서 3분 볶아요.\n3. 간장1+설탕 반스푼+다진 마늘을 넣고 섞어요.\n4. 잎채소와 ${t}를 넣고 2분만 더 볶은 뒤 불을 끄고 통깨 — 완성!`;tip=`두부는 마지막 2분만 볶아야 부드러워요.`;}
  else if(hFr){ttl=`제철 과일 샐러드`;time='10분';serv='2인분';diff='★☆☆';stp=`1. 과일을 씻어 먹기 좋게 썰어 볼에 담아요.\n2. 레몬즙1+꿀1+올리브오일 약간으로 드레싱을 만들어요.\n3. 드레싱을 과일 위에 뿌리고 살살 버무려요.\n4. 접시에 담고 민트 잎을 올려 완성!`;tip=`과일 샐러드는 먹기 직전에 드레싱을 뿌려야 물이 생기지 않아요.`;}
  else if(hFz){const fz=pick(fzs);ttl=`바삭 에어프라이어 ${fz}`;time='20분';serv='1~2인분';diff='★☆☆';stp=`1. 에어프라이어를 180도로 5분 예열해요.\n2. ${fz}을 해동 없이 바스켓에 겹치지 않게 올려요.\n3. 180도에서 8~10분 돌린 뒤 뒤집어 주세요.\n4. 200도로 올려 3분 더 — 겉바속촉 완성!`;tip=`예열과 중간 뒤집기가 핵심! 마지막에 온도를 올리면 더 바삭해요.`;}
  else if(hMi){ttl=`수제 밀크 라떼`;time='5분';serv='1인분';diff='★☆☆';stp=`1. 우유200ml를 냄비에 넣고 기포가 생길 때까지 데워요(끓이면 안돼요!).\n2. 머그잔에 데운 우유를 붓고 인스턴트 커피 1스푼을 넣어요.\n3. 시나몬 파우더를 살짝 뿌려 마무리!`;tip=`우유는 60~70도에서 불을 끄면 가장 부드러운 라떼가 완성돼요.`;}
  else{ttl=`'${names}' 활용 건강 한 끼`;time='15분';serv='1~2인분';diff='★☆☆';stp=`1. 모든 재료를 먹기 좋게 손질합니다.\n2. 팬에 올리브오일을 두르고 재료를 넣어 볶아주세요.\n3. 소금·후추로 간을 맞추고 3~5분간 익혀요.\n4. 마지막에 간장을 팬 가장자리에 둘러 향을 더하고 불을 끕니다.\n5. 접시에 담고 통깨를 뿌려 마무리!`;tip=`어떤 재료든 센 불 단시간이 핵심입니다.`;}
  return {title:ttl,time,servings:serv,difficulty:diff,steps:stp,ingredientDetails:detail,chefTip:tip,nutrition:getNutrition(group.type)};
}
function makeUsage(name) {
  const m=['삼겹살','목살','소고기','돼지고기','닭고기','닭다리','닭가슴살','불고기','차돌박이','스테이크'];
  const f=['생선','고등어','갈치','연어','참치','참치회','오징어','새우'];
  const l=['시금치','상추','깻잎','배추','양상추','로메인','케일','부추','미나리','쑥갓'];
  const r=['양파','당근','무','감자','고구마','호박','단호박'];
  const s=['버섯','느타리','팽이버섯','새송이','표고버섯'];
  const fr=['사과','청송사과','바나나','딸기','블루베리','오렌지','귤','감귤','포도','복숭아','배','키위','망고','파인애플','레몬','자몽'];
  const fz=['만두','냉동만두','냉동피자','치킨','핫도그','감자튀김','아이스크림'];
  const ca=['참치캔','통조림','햄','스팸','소시지','베이컨','라면','즉석밥','짜장','카레','레토르트','시리얼','과자','비스킷','초콜릿','사탕'];
  const gr=['쌀','잡곡','현미','밀가루','빵','식빵','베이글','파스타','국수','소면'];
  if(m.includes(name))return'150~200g, 한입 크기로 썰기';if(f.includes(name))return'1토막, 비늘·내장 제거 후 손질';
  if(l.includes(name))return'한 줌, 4cm 길이로 썰기';if(r.includes(name))return'1개, 채썰기 또는 깍둑썰기';
  if(name==='마늘'||name==='생강')return'2~3쪽, 다져서 준비';if(s.includes(name))return'1팩, 밑동 제거 후 찢기';
  if(name==='두부')return'1모, 1.5cm 두께로 썰기';if(name==='콩나물'||name==='숙주')return'한 줌, 깨끗이 씻어 물기 제거';
  if(name==='계란'||name==='달걀')return'2알, 풀어서 준비';if(name==='우유'||name==='서울우유'||name==='매일우유')return'200ml';
  if(name==='요구르트'||name==='요거트')return'1개 (85ml)';if(fz.includes(name))return'1인분, 해동 없이 그대로 사용';
  if(fr.includes(name))return'1개, 깨끗이 씻어 적당한 크기로';if(ca.includes(name))return'1캔 또는 1봉, 기름기 제거';
  if(gr.includes(name))return'1인분 (약 150g)';return'적당량';
}
function getNutrition(type) {
  const d={meat_main:{kcal:'약 450kcal',carbs:'탄수화물 22g',protein:'단백질 35g',fat:'지방 22g',note:'고기와 채소의 균형 잡힌 한 끼'},seafood:{kcal:'약 320kcal',carbs:'탄수화물 15g',protein:'단백질 28g',fat:'지방 15g',note:'오메가-3 풍부 건강식'},frozen:{kcal:'약 480kcal',carbs:'탄수화물 42g',protein:'단백질 20g',fat:'지방 24g',note:'탄수화물 비중 높은 든든한 식사'},canned:{kcal:'약 350kcal',carbs:'탄수화물 20g',protein:'단백질 25g',fat:'지방 18g',note:'나트륨 다소 높을 수 있음'},dessert:{kcal:'약 220kcal',carbs:'탄수화물 45g',protein:'단백질 6g',fat:'지방 4g',note:'비타민 가득 가벼운 디저트'},fruit_only:{kcal:'약 180kcal',carbs:'탄수화물 40g',protein:'단백질 2g',fat:'지방 2g',note:'식이섬유 풍부 저지방 간식'},grain:{kcal:'약 520kcal',carbs:'탄수화물 68g',protein:'단백질 18g',fat:'지방 18g',note:'에너지 충전용 한 그릇'},egg:{kcal:'약 280kcal',carbs:'탄수화물 8g',protein:'단백질 18g',fat:'지방 18g',note:'완전 단백질 풍부 아침 식사'},tofu_main:{kcal:'약 250kcal',carbs:'탄수화물 15g',protein:'단백질 18g',fat:'지방 12g',note:'식물성 단백질 위주 저칼로리'},misc:{kcal:'약 300kcal',carbs:'탄수화물 25g',protein:'단백질 15g',fat:'지방 15g',note:'재료에 따라 편차 있을 수 있음'},all:{kcal:'약 380kcal',carbs:'탄수화물 30g',protein:'단백질 22g',fat:'지방 18g',note:'모든 재료 활용 균형 잡힌 한 상'},};
  return d[type]||{kcal:'약 300kcal',carbs:'탄수화물 25g',protein:'단백질 15g',fat:'지방 15g',note:'추정치입니다'};
}
function buildProfileAlert() {
  const u=getSession();if(!u)return'';
  const alerts=[],h=(u.health||'').toLowerCase(),m=(u.medications||'').toLowerCase(),a=(u.allergies||'').toLowerCase();
  if(h.includes('고혈압')||h.includes('혈압'))alerts.push('고혈압: 나트륨 섭취를 줄이기 위해 간장·소금 사용량을 절반으로 줄이세요.');
  if(h.includes('당뇨')||h.includes('혈당'))alerts.push('당뇨: 탄수화물 섭취량을 확인하고 현미·통밀을 활용하세요.');
  if(h.includes('임신')||h.includes('수유'))alerts.push('임신/수유 중: 날음식·고수은 생선을 피하고 완전히 익혀 드세요.');
  if(h.includes('비건')||h.includes('채식'))alerts.push('비건/채식: 동물성 재료를 두부·콩류로 대체하세요.');
  if(h.includes('고지혈')||h.includes('콜레스테롤'))alerts.push('고지혈증: 포화지방이 많은 육류 지방 부위를 제거하세요.');
  if(h.includes('신장')||h.includes('콩팥'))alerts.push('신장 질환: 칼륨이 많은 채소와 나트륨 섭취에 주의하세요.');
  if(h.includes('글루텐'))alerts.push('글루텐 프리: 밀가루·빵·파스타 대신 쌀·메밀을 사용하세요.');
  const mw={와파린:'와파린 복용 중: 비타민K 풍부 녹색 채소 섭취량 일정하게 유지',메트포르민:'메트포르민 복용 중: 규칙적인 식사와 함께 복용',철분제:'철분제 복용 중: 칼슘이 많은 유제품과 함께 복용 금지',스타틴:'스타틴 복용 중: 자몽·자몽주스 섭취 금지'};
  for(const[k,v]of Object.entries(mw)){if(m.includes(k))alerts.push(v);}
  if(a){const ai=a.split(/[,，、\s]+/).filter(Boolean);if(ai.length>0)alerts.push(`알레르기 주의: ${ai.join(', ')} 성분 포함 여부를 확인하세요.`);}
  if(alerts.length===0)return'';
  return `<div class="banner banner-warn" style="margin-top:10px;display:block;animation:none;"><h5 style="font-size:13px;font-weight:700;color:#c33;margin-bottom:4px;">⚠️ ${u.name||u.username}님 건강 알림</h5><ul style="padding-left:16px;font-size:12px;color:#a30;line-height:1.6;">${alerts.map(a=>`<li>${a}</li>`).join('')}</ul></div>`;
}
function esc(s){return String(s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;').replace(/"/g,'&quot;');}

/* ===== 비밀번호 보기/숨기기 & CapsLock 감지 ===== */
function togglePwVisibility(inputId) {
  const inp = document.getElementById(inputId);
  if (!inp) return;
  const btn = document.getElementById(inputId + 'Toggle');
  if (inp.type === 'password') { inp.type = 'text'; if (btn) btn.textContent = '👁️'; }
  else { inp.type = 'password'; if (btn) btn.textContent = '🙈'; }
}
function detectCapsLock(e, warnId) {
  const warn = document.getElementById(warnId);
  if (!warn) return;
  warn.style.display = (e.getModifierState && e.getModifierState('CapsLock')) ? 'block' : 'none';
}
