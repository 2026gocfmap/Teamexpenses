<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>팀 지출내역 관리</title>
  
  <!-- Tailwind CSS -->
  <script src="https://cdn.tailwindcss.com"></script>
  <!-- SheetJS (Excel 처리) -->
  <script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>
  <!-- Lucide Icons -->
  <script src="https://unpkg.com/lucide@latest"></script>

  <style>
    /* 스크롤바 커스텀 */
    ::-webkit-scrollbar { width: 8px; height: 8px; }
    ::-webkit-scrollbar-track { background: #f1f5f9; }
    ::-webkit-scrollbar-thumb { background: #cbd5e1; border-radius: 4px; }
    ::-webkit-scrollbar-thumb:hover { background: #94a3b8; }
    
    .toast-enter { transform: translate(-50%, -20px); opacity: 0; }
    .toast-enter-active { transform: translate(-50%, 0); opacity: 1; transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1); }
    .toast-exit { transform: translate(-50%, 0); opacity: 1; }
    .toast-exit-active { transform: translate(-50%, -20px); opacity: 0; transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1); }
  </style>
</head>
<body class="bg-slate-50 text-slate-800 font-sans min-h-screen">

  <!-- Toast 알림 컨테이너 -->
  <div id="toast-container" class="fixed top-4 left-1/2 -translate-x-1/2 z-50 flex flex-col gap-2 pointer-events-none"></div>

  <div class="max-w-[1400px] mx-auto p-4 sm:p-6 lg:p-8">
    
    <!-- 헤더 영역 -->
    <div class="flex flex-col md:flex-row md:items-center justify-between gap-4 mb-8">
      <div>
        <h1 class="text-2xl font-bold text-slate-900 tracking-tight">팀 지출내역 관리</h1>
        <p class="text-sm text-slate-500 mt-1">업데이트 시 모든 팀원에게 실시간으로 공유됩니다.</p>
      </div>
      
      <div class="flex flex-wrap items-center gap-2">
        <button onclick="window.copyShareLink()" class="flex items-center gap-2 px-4 py-2.5 bg-slate-100 border border-slate-300 text-slate-700 rounded-lg hover:bg-slate-200 transition-colors shadow-sm text-sm font-medium">
          <i data-lucide="share-2" class="w-4 h-4"></i> 링크 복사
        </button>
        <input type="file" id="excelUpload" accept=".xlsx, .xls, .csv" class="hidden" onchange="window.handleFileUpload(event)" />
        <button onclick="document.getElementById('excelUpload').click()" id="uploadBtn" class="flex items-center gap-2 px-4 py-2.5 bg-white border border-slate-300 text-slate-700 rounded-lg hover:bg-slate-50 transition-colors shadow-sm text-sm font-medium">
          <i data-lucide="upload" class="w-4 h-4"></i> 엑셀 업로드
        </button>
        <button onclick="window.handleExport()" class="flex items-center gap-2 px-4 py-2.5 bg-white border border-slate-300 text-slate-700 rounded-lg hover:bg-slate-50 transition-colors shadow-sm text-sm font-medium">
          <i data-lucide="download" class="w-4 h-4"></i> 엑셀 다운로드
        </button>
        <button onclick="window.openModal()" class="flex items-center gap-2 px-4 py-2.5 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition-colors shadow-sm text-sm font-medium">
          <i data-lucide="plus" class="w-4 h-4"></i> 내역 추가
        </button>
      </div>
    </div>

    <!-- 요약 위젯 -->
    <div class="bg-white rounded-xl shadow-sm border border-slate-200 p-6 mb-6 flex flex-col sm:flex-row sm:items-center justify-between">
      <div>
        <h2 class="text-sm font-medium text-slate-500">총 누적 지출 금액</h2>
        <div class="text-3xl font-bold text-slate-900 mt-1">
          <span id="totalAmount">0</span><span class="text-lg text-slate-500 ml-1">원</span>
        </div>
      </div>
      <div class="mt-4 sm:mt-0 px-4 py-2 bg-blue-50 text-blue-700 rounded-lg text-sm font-medium flex items-center gap-2" id="statusBadge">
        <div class="w-2 h-2 bg-blue-500 rounded-full animate-pulse" id="statusDot"></div>
        <span id="syncStatus">서버 연결 중...</span>
      </div>
    </div>

    <!-- 데이터 테이블 -->
    <div class="bg-white rounded-xl shadow-sm border border-slate-200 overflow-hidden">
      <div class="overflow-auto max-h-[calc(100vh-280px)] min-h-[400px]">
        <table class="w-full text-left text-sm whitespace-nowrap relative">
          <thead class="text-slate-600 font-semibold sticky top-0 z-10 shadow-sm ring-1 ring-slate-200">
            <tr>
              <th class="px-4 py-3 bg-slate-50">No.</th>
              <th class="px-4 py-3 bg-slate-50">업체명</th>
              <th class="px-4 py-3 bg-slate-50 min-w-[200px]">내역</th>
              <th class="px-4 py-3 bg-slate-50">구분</th>
              <th class="px-4 py-3 bg-slate-50 text-right">금액 (원)</th>
              <th class="px-4 py-3 bg-slate-50">세금계산서</th>
              <th class="px-4 py-3 bg-slate-50">은행명</th>
              <th class="px-4 py-3 bg-slate-50">계좌번호</th>
              <th class="px-4 py-3 bg-slate-50">예금주</th>
              <th class="px-4 py-3 bg-slate-50">출금일자</th>
              <th class="px-4 py-3 bg-slate-50">비고</th>
              <th class="px-4 py-3 bg-slate-50 text-center">관리</th>
            </tr>
          </thead>
          <tbody id="tableBody" class="divide-y divide-slate-100">
            <tr>
              <td colspan="12" class="px-4 py-12 text-center text-slate-400">데이터를 불러오는 중입니다...</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>

  <!-- 추가/수정 모달 -->
  <div id="modal" class="fixed inset-0 bg-slate-900/40 backdrop-blur-sm hidden items-center justify-center z-40 p-4 opacity-0 transition-opacity">
    <div class="bg-white rounded-2xl shadow-xl w-full max-w-2xl max-h-[90vh] overflow-hidden flex flex-col transform scale-95 transition-transform" id="modalContent">
      <div class="px-6 py-4 border-b border-slate-100 flex items-center justify-between">
        <h3 class="text-lg font-semibold text-slate-900" id="modalTitle">새 지출 내역 추가</h3>
        <button onclick="window.closeModal()" class="text-slate-400 hover:text-slate-600 transition-colors">
          <i data-lucide="x" class="w-5 h-5"></i>
        </button>
      </div>
      
      <form id="dataForm" onsubmit="window.saveData(event)" class="flex-1 overflow-y-auto p-6">
        <input type="hidden" id="editId">
        <div class="grid grid-cols-1 sm:grid-cols-2 gap-4">
          <div class="space-y-1">
            <label class="text-xs font-medium text-slate-500">No.</label>
            <input type="text" id="no" class="w-full px-3 py-2 border border-slate-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 text-sm" placeholder="자동 생성">
          </div>
          <div class="space-y-1">
            <label class="text-xs font-medium text-slate-500">출금일자</label>
            <input type="date" id="date" class="w-full px-3 py-2 border border-slate-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 text-sm">
          </div>
          <div class="space-y-1 sm:col-span-2">
            <label class="text-xs font-medium text-slate-500">업체명 <span class="text-rose-500">*</span></label>
            <input type="text" id="company" required class="w-full px-3 py-2 border border-slate-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 text-sm">
          </div>
          <div class="space-y-1 sm:col-span-2">
            <label class="text-xs font-medium text-slate-500">내역 상세</label>
            <input type="text" id="details" class="w-full px-3 py-2 border border-slate-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 text-sm">
          </div>
          <div class="space-y-1">
            <label class="text-xs font-medium text-slate-500">구분</label>
            <select id="category" class="w-full px-3 py-2 border border-slate-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 text-sm">
              <option value="">선택 안함</option>
              <option value="운영">운영</option>
              <option value="기타">기타</option>
            </select>
          </div>
          <div class="space-y-1">
            <label class="text-xs font-medium text-slate-500">금액 (원) <span class="text-rose-500">*</span></label>
            <input type="number" id="amount" required class="w-full px-3 py-2 border border-slate-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 text-sm" placeholder="0">
          </div>
          <div class="space-y-1">
            <label class="text-xs font-medium text-slate-500">세금계산서</label>
            <select id="taxInvoice" class="w-full px-3 py-2 border border-slate-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 text-sm">
              <option value="">선택 안함</option>
              <option value="수신">수신</option>
              <option value="미수신">미수신</option>
            </select>
          </div>
          <div class="space-y-1">
            <label class="text-xs font-medium text-slate-500">은행명</label>
            <input type="text" id="bank" class="w-full px-3 py-2 border border-slate-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 text-sm">
          </div>
          <div class="space-y-1">
            <label class="text-xs font-medium text-slate-500">계좌번호</label>
            <input type="text" id="account" class="w-full px-3 py-2 border border-slate-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 text-sm">
          </div>
          <div class="space-y-1">
            <label class="text-xs font-medium text-slate-500">예금주</label>
            <input type="text" id="holder" class="w-full px-3 py-2 border border-slate-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 text-sm">
          </div>
          <div class="space-y-1 sm:col-span-2">
            <label class="text-xs font-medium text-slate-500">비고</label>
            <input type="text" id="note" class="w-full px-3 py-2 border border-slate-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 text-sm">
          </div>
        </div>
      </form>
      
      <div class="px-6 py-4 border-t border-slate-100 bg-slate-50 flex justify-end gap-3">
        <button type="button" onclick="window.closeModal()" class="px-4 py-2 text-sm font-medium text-slate-600 hover:text-slate-800 transition-colors">취소</button>
        <button type="submit" form="dataForm" class="flex items-center gap-2 px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition-colors text-sm font-medium shadow-sm">
          <i data-lucide="save" class="w-4 h-4"></i> 저장하기
        </button>
      </div>
    </div>
  </div>

  <!-- Firebase 및 애플리케이션 로직 -->
  <script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/10.10.0/firebase-app.js";
    import { getAuth, signInAnonymously } from "https://www.gstatic.com/firebasejs/10.10.0/firebase-auth.js";
    import { getFirestore, collection, onSnapshot, addDoc, deleteDoc, doc, updateDoc } from "https://www.gstatic.com/firebasejs/10.10.0/firebase-firestore.js";

    // =========================================================================
    // 사용자님의 Firebase 설정
    // =========================================================================
    const firebaseConfig = {
      apiKey: "AIzaSyAL3hN-HxDsYnJlzlda2C2lxWP_35gkeVw",
      authDomain: "teamexpenses-d5d15.firebaseapp.com",
      projectId: "teamexpenses-d5d15",
      storageBucket: "teamexpenses-d5d15.firebasestorage.app",
      messagingSenderId: "355994869950",
      appId: "1:355994869950:web:19223849b77e9a08de3eef",
      measurementId: "G-887CTM7QDK"
    };

    // Firebase 초기화
    let app, auth, db;
    try {
      app = initializeApp(firebaseConfig);
      auth = getAuth(app);
      db = getFirestore(app);
    } catch (e) {
      console.error("Firebase 초기화 에러:", e);
      document.getElementById('syncStatus').innerText = "데이터베이스 설정 필요";
      document.getElementById('statusBadge').className = "mt-4 sm:mt-0 px-4 py-2 bg-rose-50 text-rose-700 rounded-lg text-sm font-medium flex items-center gap-2";
      document.getElementById('statusDot').className = "w-2 h-2 bg-rose-500 rounded-full";
    }

    let globalExpenses = [];
    const COLLECTION_NAME = "team_expenses";

    // 아이콘 렌더링
    lucide.createIcons();

    // 알림 시스템 (Toast)
    window.showToast = (message, type = 'success') => {
      const container = document.getElementById('toast-container');
      const toast = document.createElement('div');
      const isSuccess = type === 'success';
      toast.className = `flex items-center gap-2 px-4 py-3 rounded-lg shadow-lg text-white font-medium pointer-events-auto toast-enter ${isSuccess ? 'bg-emerald-600' : 'bg-rose-600'}`;
      toast.innerHTML = `<i data-lucide="${isSuccess ? 'check-circle-2' : 'alert-circle'}" class="w-5 h-5"></i><span>${message}</span>`;
      container.appendChild(toast);
      lucide.createIcons({ root: toast });
      
      requestAnimationFrame(() => {
        toast.classList.remove('toast-enter');
        toast.classList.add('toast-enter-active');
      });

      setTimeout(() => {
        toast.classList.remove('toast-enter-active');
        toast.classList.add('toast-exit-active');
        setTimeout(() => toast.remove(), 300);
      }, 3000);
    };

    // 데이터베이스 실시간 구독 설정 함수
    const setupDatabaseListeners = () => {
      const expensesRef = collection(db, COLLECTION_NAME);
      
      onSnapshot(expensesRef, (snapshot) => {
        globalExpenses = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
        renderTable();
        
        document.getElementById('syncStatus').innerText = "실시간 동기화 중";
        document.getElementById('statusBadge').className = "mt-4 sm:mt-0 px-4 py-2 bg-blue-50 text-blue-700 rounded-lg text-sm font-medium flex items-center gap-2";
        document.getElementById('statusDot').className = "w-2 h-2 bg-blue-500 rounded-full animate-pulse";
      }, (error) => {
        console.error("Firestore 접근 권한 에러:", error);
        
        document.getElementById('syncStatus').innerText = "연결 오류 (권한 부족)";
        document.getElementById('statusBadge').className = "mt-4 sm:mt-0 px-4 py-2 bg-rose-50 text-rose-700 rounded-lg text-sm font-medium flex items-center gap-2";
        document.getElementById('statusDot').className = "w-2 h-2 bg-rose-500 rounded-full";
        
        // 권한 에러 발생 시 사용자에게 명확한 해결 방법을 화면에 표시합니다.
        document.getElementById('tableBody').innerHTML = `
          <tr>
            <td colspan="12" class="px-4 py-12 text-center">
              <div class="inline-block text-left bg-rose-50 p-6 rounded-xl border border-rose-200 shadow-sm max-w-2xl w-full">
                <div class="flex items-center gap-2 text-rose-600 font-bold text-lg mb-2">
                  <i data-lucide="alert-triangle" class="w-6 h-6"></i>
                  데이터베이스 접근 권한 에러
                </div>
                <p class="text-sm text-slate-700 mb-4">파이어베이스 보안 규칙이 막혀 있어 데이터를 읽고 쓸 수 없습니다.</p>
                <div class="bg-white p-4 rounded-lg border border-slate-200">
                  <p class="font-bold text-slate-900 mb-3 text-sm">✅ 이렇게 해결하세요:</p>
                  <ol class="list-decimal pl-5 space-y-2 text-sm text-slate-600">
                    <li><a href="https://console.firebase.google.com/" target="_blank" class="text-blue-600 hover:underline">Firebase 콘솔</a>에 접속합니다.</li>
                    <li>좌측 메뉴에서 <b>Firestore Database</b>를 클릭합니다.</li>
                    <li>화면 상단의 <b>[규칙(Rules)]</b> 탭을 클릭합니다.</li>
                    <li>아래 코드를 복사하여 화면의 기존 내용을 모두 지우고 붙여넣습니다:
                      <pre class="bg-slate-50 p-3 mt-2 rounded border border-slate-200 text-xs text-slate-800 font-mono overflow-x-auto">rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read, write: if true;
    }
  }
}</pre>
                    </li>
                    <li>우측 상단의 <b>[게시(Publish)]</b> 버튼을 누른 뒤 이 페이지를 새로고침 하세요.</li>
                  </ol>
                </div>
              </div>
            </td>
          </tr>
        `;
        lucide.createIcons();
      });
    };

    // 1. 익명 로그인 시도 -> 2. 성공/실패 여부 관계없이 DB 연결 시도
    if (auth && db) {
      signInAnonymously(auth)
        .then(() => {
          console.log("익명 로그인 성공");
          setupDatabaseListeners();
        })
        .catch((error) => {
          console.warn("익명 로그인이 비활성화 되어있습니다. 인증 없이 접근을 시도합니다.", error);
          setupDatabaseListeners(); // 인증 실패 시에도 연결 시도
        });
    }

    // 통화 포맷
    const formatCurrency = (amount) => new Intl.NumberFormat('ko-KR').format(amount || 0);

    // 테이블 렌더링
    const renderTable = () => {
      const tbody = document.getElementById('tableBody');
      const sorted = [...globalExpenses].sort((a, b) => (Number(a.no) || 0) - (Number(b.no) || 0));
      
      let total = 0;
      let html = '';

      if (sorted.length === 0) {
        html = `<tr><td colspan="12" class="px-4 py-12 text-center text-slate-500">등록된 지출 내역이 없습니다. (엑셀을 업로드 하거나 내역을 추가해 보세요)</td></tr>`;
      } else {
        sorted.forEach((item, index) => {
          total += Number(item.amount) || 0;
          
          html += `
            <tr class="hover:bg-slate-50 transition-colors">
              <td class="px-4 py-3">
                <div class="flex items-center gap-2">
                  <div class="flex flex-col">
                    <button onclick="window.moveItem(${index}, 'up')" ${index === 0 ? 'disabled' : ''} class="text-slate-300 hover:text-blue-600 disabled:opacity-20 transition-colors"><i data-lucide="arrow-up" class="w-4 h-4"></i></button>
                    <button onclick="window.moveItem(${index}, 'down')" ${index === sorted.length - 1 ? 'disabled' : ''} class="text-slate-300 hover:text-blue-600 disabled:opacity-20 transition-colors"><i data-lucide="arrow-down" class="w-4 h-4"></i></button>
                  </div>
                  <span class="font-medium text-slate-700 w-4 text-center">${item.no || ''}</span>
                </div>
              </td>
              <td class="px-4 py-3 font-medium text-slate-900">${item.company || ''}</td>
              <td class="px-4 py-3 truncate max-w-[250px]" title="${item.details || ''}">${item.details || ''}</td>
              <td class="px-4 py-3"><span class="px-2 py-1 bg-slate-100 text-slate-600 rounded text-xs font-medium">${item.category || ''}</span></td>
              <td class="px-4 py-3 text-right font-medium text-slate-900">${formatCurrency(item.amount)}</td>
              <td class="px-4 py-3">${item.taxInvoice || ''}</td>
              <td class="px-4 py-3">${item.bank || ''}</td>
              <td class="px-4 py-3 text-slate-500">${item.account || ''}</td>
              <td class="px-4 py-3 text-slate-700">${item.holder || ''}</td>
              <td class="px-4 py-3">${item.date || ''}</td>
              <td class="px-4 py-3 truncate max-w-[150px]" title="${item.note || ''}">${item.note || ''}</td>
              <td class="px-4 py-3 text-center">
                <div class="flex justify-center items-center gap-2">
                  <button onclick="window.editItem('${item.id}')" class="p-1.5 text-slate-400 hover:text-blue-600 hover:bg-blue-50 rounded"><i data-lucide="edit-2" class="w-4 h-4"></i></button>
                  <button onclick="window.deleteItem('${item.id}')" class="p-1.5 text-slate-400 hover:text-rose-600 hover:bg-rose-50 rounded"><i data-lucide="trash-2" class="w-4 h-4"></i></button>
                </div>
              </td>
            </tr>
          `;
        });
      }

      tbody.innerHTML = html;
      document.getElementById('totalAmount').innerText = formatCurrency(total);
      lucide.createIcons({ root: tbody });
    };

    // 데이터 저장 (추가/수정)
    window.saveData = async (e) => {
      e.preventDefault();
      if (!db) return window.showToast('데이터베이스가 연결되지 않았습니다.', 'error');

      const id = document.getElementById('editId').value;
      const amount = Number(document.getElementById('amount').value) || 0;
      
      let no = document.getElementById('no').value;
      if (!no && !id) {
        const maxNo = globalExpenses.reduce((max, item) => {
          const num = Number(item.no);
          return !isNaN(num) && num > max ? num : max;
        }, 0);
        no = String(maxNo + 1);
      }

      const data = {
        no, amount,
        company: document.getElementById('company').value,
        details: document.getElementById('details').value,
        category: document.getElementById('category').value,
        taxInvoice: document.getElementById('taxInvoice').value,
        bank: document.getElementById('bank').value,
        account: document.getElementById('account').value,
        holder: document.getElementById('holder').value,
        date: document.getElementById('date').value,
        note: document.getElementById('note').value,
        updatedAt: new Date().toISOString()
      };

      try {
        if (id) {
          await updateDoc(doc(db, COLLECTION_NAME, id), data);
          window.showToast('수정되었습니다.');
        } else {
          data.createdAt = new Date().toISOString();
          await addDoc(collection(db, COLLECTION_NAME), data);
          window.showToast('추가되었습니다.');
        }
        window.closeModal();
      } catch (err) {
        window.showToast('권한이 없어 저장할 수 없습니다.', 'error');
        console.error(err);
      }
    };

    // 삭제
    window.deleteItem = async (id) => {
      if (!confirm('정말 삭제하시겠습니까?')) return;
      try {
        await deleteDoc(doc(db, COLLECTION_NAME, id));
        window.showToast('삭제되었습니다.');
      } catch (err) {
        window.showToast('권한이 없어 삭제할 수 없습니다.', 'error');
      }
    };

    // 순서 이동
    window.moveItem = async (index, direction) => {
      const sorted = [...globalExpenses].sort((a, b) => (Number(a.no) || 0) - (Number(b.no) || 0));
      const newIndex = direction === 'up' ? index - 1 : index + 1;
      
      const currentItem = sorted[index];
      const targetItem = sorted[newIndex];
      if (!currentItem || !targetItem) return;

      try {
        await updateDoc(doc(db, COLLECTION_NAME, currentItem.id), { no: targetItem.no });
        await updateDoc(doc(db, COLLECTION_NAME, targetItem.id), { no: currentItem.no });
      } catch (err) {
        window.showToast('순서 변경 실패', 'error');
      }
    };

    // 엑셀 업로드
    window.handleFileUpload = async (e) => {
      const file = e.target.files[0];
      if (!file) return;

      try {
        const reader = new FileReader();
        reader.onload = async (event) => {
          const data = new Uint8Array(event.target.result);
          const workbook = XLSX.read(data, { type: 'array' });
          const worksheet = workbook.Sheets[workbook.SheetNames[0]];
          const jsonData = XLSX.utils.sheet_to_json(worksheet, { header: 1 });

          let headerRowIndex = jsonData.findIndex(row => row && row.includes('No.') && row.includes('업체명'));
          if (headerRowIndex === -1) return window.showToast('지원하지 않는 양식입니다.', 'error');

          const headers = jsonData[headerRowIndex];
          const rows = jsonData.slice(headerRowIndex + 1);

          const expensesToAdd = [];
          let duplicateCount = 0;

          rows.forEach(row => {
            const getVal = col => row[headers.indexOf(col)] || '';
            const item = {
              no: String(getVal('No.')),
              company: getVal('업체명'),
              details: getVal('내역'),
              category: getVal('구분'),
              amount: Number(getVal('금액')) || 0,
              taxInvoice: getVal('세금계산서'),
              bank: getVal('은행명'),
              account: getVal('계좌번호'),
              holder: getVal('예금주'),
              date: getVal('출금일자'),
              note: getVal('비고'),
              createdAt: new Date().toISOString()
            };

            if (!item.company && !item.amount) return;

            const isDuplicate = globalExpenses.some(ex => 
              ex.company === item.company && ex.details === item.details &&
              Number(ex.amount) === item.amount && ex.date === item.date
            );

            if (isDuplicate) duplicateCount++;
            else expensesToAdd.push(item);
          });

          for (const item of expensesToAdd) await addDoc(collection(db, COLLECTION_NAME), item);

          let msg = expensesToAdd.length > 0 ? `${expensesToAdd.length}건 추가됨.` : '추가할 새 데이터가 없습니다.';
          if (duplicateCount > 0) msg += ` (중복 ${duplicateCount}건 제외)`;
          window.showToast(msg, expensesToAdd.length > 0 ? 'success' : 'info');
          e.target.value = '';
        };
        reader.readAsArrayBuffer(file);
      } catch (err) {
        window.showToast('권한 문제로 업로드할 수 없습니다.', 'error');
      }
    };

    // 엑셀 다운로드
    window.handleExport = () => {
      const sorted = [...globalExpenses].sort((a, b) => (Number(a.no) || 0) - (Number(b.no) || 0));
      const exportData = sorted.map(item => ({
        'No.': item.no, '업체명': item.company, '내역': item.details, '구분': item.category,
        '금액': item.amount, '세금계산서': item.taxInvoice, '은행명': item.bank,
        '계좌번호': item.account, '예금주': item.holder, '출금일자': item.date, '비고': item.note
      }));
      const ws = XLSX.utils.json_to_sheet(exportData);
      const wb = XLSX.utils.book_new();
      XLSX.utils.book_append_sheet(wb, ws, "지출내역");
      XLSX.writeFile(wb, `팀지출내역_${new Date().toISOString().split('T')[0]}.xlsx`);
      window.showToast('엑셀이 다운로드되었습니다.');
    };

    // 모달 제어
    const modal = document.getElementById('modal');
    const modalContent = document.getElementById('modalContent');
    
    window.openModal = () => {
      document.getElementById('dataForm').reset();
      document.getElementById('editId').value = '';
      document.getElementById('modalTitle').innerText = '새 지출 내역 추가';
      
      modal.classList.remove('hidden');
      modal.classList.add('flex');
      requestAnimationFrame(() => {
        modal.classList.remove('opacity-0');
        modalContent.classList.remove('scale-95');
      });
    };

    window.editItem = (id) => {
      const item = globalExpenses.find(ex => ex.id === id);
      if(!item) return;
      
      document.getElementById('editId').value = item.id;
      document.getElementById('no').value = item.no || '';
      document.getElementById('company').value = item.company || '';
      document.getElementById('details').value = item.details || '';
      document.getElementById('category').value = item.category || '';
      document.getElementById('amount').value = item.amount || '';
      document.getElementById('taxInvoice').value = item.taxInvoice || '';
      document.getElementById('bank').value = item.bank || '';
      document.getElementById('account').value = item.account || '';
      document.getElementById('holder').value = item.holder || '';
      document.getElementById('date').value = item.date || '';
      document.getElementById('note').value = item.note || '';
      
      document.getElementById('modalTitle').innerText = '지출 내역 수정';
      modal.classList.remove('hidden');
      modal.classList.add('flex');
      requestAnimationFrame(() => {
        modal.classList.remove('opacity-0');
        modalContent.classList.remove('scale-95');
      });
    };

    window.closeModal = () => {
      modal.classList.add('opacity-0');
      modalContent.classList.add('scale-95');
      setTimeout(() => {
        modal.classList.remove('flex');
        modal.classList.add('hidden');
      }, 300);
    };

    window.copyShareLink = () => {
      navigator.clipboard.writeText(window.location.href).then(() => {
        window.showToast('링크가 복사되었습니다.');
      }).catch(() => {
        window.showToast('주소창의 링크를 직접 복사해주세요.', 'error');
      });
    };
  </script>
</body>
</html>
