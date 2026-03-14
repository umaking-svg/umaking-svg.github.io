import React, { useState, useEffect } from 'react';
import { initializeApp } from 'firebase/app';
import { 
  getAuth, 
  signInAnonymously, 
  signInWithCustomToken,
  onAuthStateChanged 
} from 'firebase/auth';
import { 
  getFirestore, 
  collection, 
  doc, 
  setDoc, 
  getDoc, 
  getDocs, 
  onSnapshot
} from 'firebase/firestore';
import { 
  Shield, Wand2, Sword, Target, Cross, 
  Users, Lock, LogOut, BarChart3, X, Key 
} from 'lucide-react';

/**
 * [환경 설정 가이드]
 * 1. 미리보기 환경: 시스템이 제공하는 __firebase_config를 자동으로 사용합니다.
 * 2. 외부 배포(GitHub Pages): 아래 firebaseConfig 주석을 해제하고 본인의 키를 입력하세요.
 */
const firebaseConfig = typeof __firebase_config !== 'undefined' 
  ? JSON.parse(__firebase_config) 
  : {
      apiKey: "YOUR_API_KEY",
      authDomain: "YOUR_PROJECT.firebaseapp.com",
      projectId: "YOUR_PROJECT_ID",
      storageBucket: "YOUR_PROJECT.appspot.com",
      messagingSenderId: "YOUR_SENDER_ID",
      appId: "YOUR_APP_ID"
    };

const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'rpg-event-2024';

const JOBS = [
  { id: 'warrior', name: '전사', color: 'bg-red-500', icon: Shield, desc: '강력한 체력으로 동료를 지키는 방패' },
  { id: 'mage', name: '마법사', color: 'bg-blue-500', icon: Wand2, desc: '강력한 원소 마법을 구사하는 현자' },
  { id: 'rogue', name: '도적', color: 'bg-purple-600', icon: Sword, desc: '그림자 속에서 적을 기습하는 암살자' },
  { id: 'archer', name: '궁수', color: 'bg-green-600', icon: Target, desc: '백발백중의 실력을 자랑하는 명사수' },
  { id: 'priest', name: '사제', color: 'bg-yellow-500', icon: Cross, desc: '신성한 힘으로 상처를 치유하는 인도자' }
];

const ADMIN_PASSWORD = "admin777"; 

export default function App() {
  const [user, setUser] = useState(null);
  const [nickname, setNickname] = useState('');
  const [password, setPassword] = useState('');
  const [loading, setLoading] = useState(true);
  const [assignment, setAssignment] = useState(null);
  const [error, setError] = useState('');
  const [view, setView] = useState('login'); 
  const [adminInput, setAdminInput] = useState('');
  const [allUsers, setAllUsers] = useState([]);
  const [stats, setStats] = useState({});

  // Mandatory Rule 1: Strict Paths
  const getCollectionRef = () => {
    return collection(db, 'artifacts', appId, 'public', 'data', 'registrations');
  };

  const getDocRef = (id) => {
    return doc(db, 'artifacts', appId, 'public', 'data', 'registrations', id);
  };

  // Mandatory Rule 3: Auth before queries
  useEffect(() => {
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInWithCustomToken(auth, __initial_auth_token);
        } else {
          await signInAnonymously(auth);
        }
      } catch (err) {
        console.error("인증 에러:", err);
      }
    };
    initAuth();

    const unsubscribe = onAuthStateChanged(auth, (u) => {
      setUser(u);
      setLoading(false);
    });
    return () => unsubscribe();
  }, []);

  // Admin view listener with auth guard
  useEffect(() => {
    if (!user || view !== 'admin') return;

    const usersRef = getCollectionRef();
    const unsubscribe = onSnapshot(usersRef, (snapshot) => {
      const data = snapshot.docs.map(doc => doc.data());
      const sortedData = data.sort((a, b) => (b.timestamp || 0) - (a.timestamp || 0));
      setAllUsers(sortedData);
      
      const counts = {};
      JOBS.forEach(j => counts[j.name] = 0);
      data.forEach(u => {
        if (u.jobName) counts[u.jobName] = (counts[u.jobName] || 0) + 1;
      });
      setStats(counts);
    }, (err) => {
      console.error("Firestore 감시 에러:", err);
    });

    return () => unsubscribe();
  }, [user, view]);

  const assignJobBalanced = async () => {
    const usersRef = getCollectionRef();
    const allSnap = await getDocs(usersRef);
    const currentData = allSnap.docs.map(d => d.data());
    
    const counts = {};
    JOBS.forEach(j => counts[j.name] = 0);
    currentData.forEach(u => {
      if (u.jobName) counts[u.jobName] = (counts[u.jobName] || 0) + 1;
    });

    const minCount = Math.min(...Object.values(counts));
    const leastJobs = JOBS.filter(j => counts[j.name] === minCount);
    return leastJobs[Math.floor(Math.random() * leastJobs.length)];
  };

  const handleJoin = async (e) => {
    e.preventDefault();
    if (!user) { setError('인증 중입니다. 잠시 후 시도해주세요.'); return; }
    if (!nickname.trim() || !password.trim()) { setError('닉네임과 비밀번호를 입력해주세요.'); return; }

    setError('');
    setLoading(true);

    try {
      const docRef = getDocRef(nickname);
      const docSnap = await getDoc(docRef);

      if (docSnap.exists()) {
        const data = docSnap.data();
        if (data.password === password) {
          setAssignment(data);
          setView('result');
        } else {
          setError('이미 등록된 이름입니다. 비밀번호가 다릅니다.');
        }
      } else {
        const selectedJob = await assignJobBalanced();
        
        const newAssignment = {
          nickname,
          password,
          jobId: selectedJob.id,
          jobName: selectedJob.name,
          timestamp: Date.now()
        };

        await setDoc(docRef, newAssignment);
        setAssignment(newAssignment);
        setView('result');
      }
    } catch (err) {
      console.error(err);
      setError('서버 통신 중 오류가 발생했습니다.');
    } finally {
      setLoading(false);
    }
  };

  const handleAdminAuth = (e) => {
    e.preventDefault();
    if (adminInput === ADMIN_PASSWORD) {
      setError('');
      setView('admin');
      setAdminInput('');
    } else {
      setError('관리자 비밀번호가 틀렸습니다.');
    }
  };

  if (loading && !user) {
    return (
      <div className="min-h-screen bg-slate-950 flex flex-col items-center justify-center text-white p-6 font-sans">
        <div className="animate-spin w-10 h-10 border-4 border-indigo-500 border-t-transparent rounded-full mb-4"></div>
        <p className="animate-pulse">서버에 접속하고 있습니다...</p>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-slate-950 text-slate-100 font-sans p-4 sm:p-8 selection:bg-indigo-500/30">
      <div className="max-w-md mx-auto">
        <header className="text-center mb-10 pt-4 animate-in fade-in slide-in-from-top-4 duration-700">
          <h1 className="text-4xl font-black bg-gradient-to-br from-indigo-400 via-purple-400 to-pink-500 bg-clip-text text-transparent tracking-tighter">
            QUEST MASTER
          </h1>
          <p className="text-slate-500 mt-2 font-medium">RPG 직업 자동 배정 시스템</p>
        </header>

        {view === 'login' && (
          <div className="bg-slate-900/50 backdrop-blur-xl border border-white/10 rounded-3xl p-8 shadow-2xl animate-in zoom-in-95 duration-500">
            <div className="flex items-center gap-3 mb-8">
              <div className="w-10 h-10 bg-indigo-500/20 rounded-xl flex items-center justify-center">
                <Users className="w-6 h-6 text-indigo-400" />
              </div>
              <div>
                <h2 className="text-xl font-bold">참가자 등록</h2>
                <p className="text-xs text-slate-500">운명의 직업을 확인하세요</p>
              </div>
            </div>
            
            <form onSubmit={handleJoin} className="space-y-4">
              <div className="space-y-1">
                <label className="text-xs font-bold text-slate-500 ml-1">이름(닉네임)</label>
                <input
                  type="text"
                  value={nickname}
                  onChange={(e) => setNickname(e.target.value)}
                  className="w-full bg-slate-800/50 border border-slate-700 rounded-2xl px-5 py-4 outline-none focus:ring-2 focus:ring-indigo-500 transition-all placeholder:text-slate-600"
                  placeholder="예: 안시헌"
                />
              </div>
              <div className="space-y-1">
                <label className="text-xs font-bold text-slate-500 ml-1">비밀번호</label>
                <input
                  type="password"
                  value={password}
                  onChange={(e) => setPassword(e.target.value)}
                  className="w-full bg-slate-800/50 border border-slate-700 rounded-2xl px-5 py-4 outline-none focus:ring-2 focus:ring-indigo-500 transition-all placeholder:text-slate-600"
                  placeholder="추후 본인 확인용"
                />
              </div>
              
              {error && <div className="bg-red-500/10 border border-red-500/20 text-red-400 p-4 rounded-xl text-sm">{error}</div>}
              
              <button type="submit" disabled={loading} className="w-full bg-indigo-600 hover:bg-indigo-500 disabled:opacity-50 text-white font-black py-4 rounded-2xl shadow-xl shadow-indigo-500/20 transition-all active:scale-[0.98] mt-2">
                {loading ? '배정 중...' : '직업 결정하기'}
              </button>
            </form>

            <button 
              onClick={() => { setView('admin_auth'); setError(''); }}
              className="mt-10 w-full text-slate-600 hover:text-slate-400 text-[10px] flex items-center justify-center gap-1 font-bold tracking-widest transition-colors"
            >
              <Lock className="w-3 h-3" /> 관리자 전용 메뉴
            </button>
          </div>
        )}

        {view === 'admin_auth' && (
          <div className="bg-slate-900 border border-white/5 rounded-3xl p-8 shadow-2xl animate-in fade-in zoom-in duration-300">
            <div className="flex justify-between items-center mb-8">
              <h2 className="text-xl font-bold text-yellow-500 flex items-center gap-2">
                <Key className="w-6 h-6" /> 관리자 로그인
              </h2>
              <button onClick={() => setView('login')} className="text-slate-500 hover:text-white transition-colors">
                <X className="w-6 h-6" />
              </button>
            </div>
            <form onSubmit={handleAdminAuth} className="space-y-4">
              <input
                type="password"
                autoFocus
                value={adminInput}
                onChange={(e) => setAdminInput(e.target.value)}
                className="w-full bg-slate-800 border border-slate-700 rounded-2xl px-5 py-4 outline-none focus:ring-2 focus:ring-yellow-500"
                placeholder="관리자 코드를 입력하세요"
              />
              {error && <p className="text-red-400 text-sm font-medium">{error}</p>}
              <button type="submit" className="w-full bg-yellow-600 hover:bg-yellow-500 text-white font-black py-4 rounded-2xl transition-all active:scale-[0.98]">
                현황판 접속
              </button>
            </form>
          </div>
        )}

        {view === 'result' && assignment && (
          <div className="space-y-6 animate-in zoom-in duration-500">
            <div className="bg-slate-900 border border-white/10 rounded-[2.5rem] overflow-hidden shadow-2xl">
              <div className={`${JOBS.find(j => j.id === assignment.jobId)?.color} p-12 text-center relative overflow-hidden`}>
                <div className="absolute top-0 right-0 w-32 h-32 bg-white/10 blur-3xl rounded-full -mr-16 -mt-16"></div>
                <div className="bg-white/20 w-28 h-28 rounded-full flex items-center justify-center mx-auto mb-6 border border-white/40 backdrop-blur-lg shadow-inner">
                  {React.createElement(JOBS.find(j => j.id === assignment.jobId)?.icon || Shield, { className: "w-14 h-14 text-white" })}
                </div>
                <p className="text-white/70 text-sm font-bold tracking-widest mb-1 uppercase">Adventurer Class</p>
                <h2 className="text-6xl font-black text-white tracking-tighter drop-shadow-md">
                  {assignment.jobName}
                </h2>
              </div>
              <div className="p-10 text-center bg-gradient-to-b from-slate-900 to-slate-950">
                <p className="text-indigo-400 font-bold mb-2">{assignment.nickname} 용사님,</p>
                <p className="text-slate-300 italic mb-10 text-lg leading-relaxed px-2">
                  "{JOBS.find(j => j.id === assignment.jobId)?.desc}"
                </p>
                <button onClick={() => { setView('login'); setNickname(''); setPassword(''); }} className="text-slate-500 hover:text-white transition-colors flex items-center justify-center gap-2 mx-auto font-bold text-sm">
                  <LogOut className="w-4 h-4" /> 로그아웃
                </button>
              </div>
            </div>
          </div>
        )}

        {view === 'admin' && (
          <div className="max-w-xl mx-auto space-y-4 animate-in slide-in-from-bottom-6 duration-700">
            <div className="flex justify-between items-center bg-slate-900 p-5 rounded-2xl border border-white/5">
              <h2 className="font-black flex items-center gap-3 text-lg">
                <BarChart3 className="w-6 h-6 text-indigo-400" /> 현황 마스터
              </h2>
              <button onClick={() => setView('login')} className="text-xs bg-slate-800 hover:bg-slate-700 px-4 py-2 rounded-xl transition-colors font-bold border border-slate-700">종료</button>
            </div>

            <div className="grid grid-cols-2 sm:grid-cols-3 gap-3">
              <div className="bg-indigo-600 p-4 rounded-3xl shadow-lg shadow-indigo-600/20 text-center col-span-2 sm:col-span-1">
                <div className="text-4xl font-black text-white">{allUsers.length}</div>
                <div className="text-[10px] uppercase font-black text-indigo-200 tracking-tighter">Total Players</div>
              </div>
              {JOBS.map(job => (
                <div key={job.id} className="bg-slate-900 border border-white/5 p-4 rounded-3xl text-center">
                  <div className="text-2xl font-black text-slate-100">{stats[job.name] || 0}</div>
                  <div className="text-[10px] font-bold text-slate-500 uppercase">{job.name}</div>
                </div>
              ))}
            </div>

            <div className="bg-slate-900 border border-white/5 rounded-[2rem] overflow-hidden shadow-xl">
              <div className="p-5 border-b border-white/5 bg-slate-800/30 flex justify-between items-center">
                <h3 className="text-xs font-black text-slate-400 uppercase tracking-widest">모험가 실시간 리스트</h3>
                <span className="text-[10px] text-slate-600">Total: {allUsers.length}</span>
              </div>
              <div className="max-h-[450px] overflow-y-auto scrollbar-hide font-sans">
                <table className="w-full">
                  <thead className="bg-slate-900/80 sticky top-0 text-slate-500 text-[10px] font-black uppercase tracking-widest border-b border-white/5">
                    <tr>
                      <th className="px-6 py-4 text-left">이름</th>
                      <th className="px-6 py-4 text-right">직업</th>
                    </tr>
                  </thead>
                  <tbody className="divide-y divide-white/5">
                    {allUsers.length === 0 ? (
                      <tr><td colSpan="2" className="p-16 text-center text-slate-600 font-medium italic">아직 등록된 모험가가 없습니다.</td></tr>
                    ) : (
                      allUsers.map((u, i) => (
                        <tr key={i} className="hover:bg-indigo-500/5 transition-colors group">
                          <td className="px-6 py-4 font-bold text-slate-200 group-hover:text-white">{u.nickname}</td>
                          <td className="px-6 py-4 text-right">
                            <span className={`px-3 py-1 rounded-full text-[10px] font-black text-white shadow-sm ${JOBS.find(j => j.id === u.jobId)?.color}`}>
                              {u.jobName}
                            </span>
                          </td>
                        </tr>
                      ))
                    )}
                  </tbody>
                </table>
              </div>
            </div>
          </div>
        )}
      </div>

      <footer className="mt-16 text-center text-slate-700 text-[9px] font-bold tracking-[0.3em] pb-12 opacity-50 uppercase">
        &mdash; Recreation Guild Management v1.0 &mdash;
      </footer>
    </div>
  );
}
