<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>QUEST MASTER - 직업 배정 시스템</title>
    <!-- Tailwind CSS CDN -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- React & Babel CDN (단일 파일 실행용) -->
    <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <!-- Lucide Icons -->
    <script src="https://unpkg.com/lucide@latest"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Pretendard:wght@400;700;900&display=swap');
        body { font-family: 'Pretendard', sans-serif; }
        .scrollbar-hide::-webkit-scrollbar { display: none; }
        
        /* 애니메이션 효과 */
        .fade-in { animation: fadeIn 0.5s ease-out forwards; }
        @keyframes fadeIn {
            from { opacity: 0; transform: translateY(10px); }
            to { opacity: 1; transform: translateY(0); }
        }
    </style>
</head>
<body class="bg-slate-950 text-slate-100">
    <div id="root"></div>

    <!-- data-type="module"을 추가하여 Babel이 import 구문을 require()로 변환하지 않도록 설정 -->
    <script type="text/babel" data-type="module">
        const { useState, useEffect, useRef } = React;

        // --- Firebase Modules (ESM via CDN) ---
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, collection, doc, setDoc, getDoc, getDocs, onSnapshot } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Firebase Config
        const firebaseConfig = typeof __firebase_config !== 'undefined' 
            ? JSON.parse(__firebase_config) 
            : {
                apiKey: "", // 환경 변수에서 제공됨
                authDomain: "",
                projectId: "",
                storageBucket: "",
                messagingSenderId: "",
                appId: ""
            };

        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'rpg-event-2024';

        const JOBS = [
            { id: 'warrior', name: '전사', color: 'bg-red-500', icon: 'shield', desc: '강력한 체력으로 동료를 지키는 방패' },
            { id: 'mage', name: '마법사', color: 'bg-blue-500', icon: 'wand-2', desc: '강력한 원소 마법을 구사하는 현자' },
            { id: 'rogue', name: '도적', color: 'bg-purple-600', icon: 'sword', desc: '그림자 속에서 적을 기습하는 암살자' },
            { id: 'archer', name: '궁수', color: 'bg-green-600', icon: 'target', desc: '백발백중의 실력을 자랑하는 명사수' },
            { id: 'priest', name: '사제', color: 'bg-yellow-500', icon: 'cross', desc: '신성한 힘으로 상처를 치유하는 인도자' }
        ];

        const ADMIN_PASSWORD = "admin777"; 

        // 아이콘 컴포넌트
        const Icon = ({ name, className }) => {
            const iconRef = useRef(null);
            useEffect(() => {
                if (window.lucide) {
                    window.lucide.createIcons();
                }
            }, [name]);
            return <i data-lucide={name} className={className} ref={iconRef}></i>;
        };

        function App() {
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

            // 인증 처리 (Rule 3)
            useEffect(() => {
                const initAuth = async () => {
                    try {
                        await signInAnonymously(auth);
                    } catch (err) {
                        console.error("Auth error:", err);
                    }
                };
                initAuth();
                const unsubscribe = onAuthStateChanged(auth, (u) => {
                    setUser(u);
                    setLoading(false);
                });
                return () => unsubscribe();
            }, []);

            // 관리자 데이터 감시 (Rule 1)
            useEffect(() => {
                if (!user || view !== 'admin') return;
                const usersRef = collection(db, 'artifacts', appId, 'public', 'data', 'registrations');
                const unsubscribe = onSnapshot(usersRef, (snapshot) => {
                    const data = snapshot.docs.map(doc => doc.data());
                    setAllUsers(data.sort((a, b) => (b.timestamp || 0) - (a.timestamp || 0)));
                    const counts = {};
                    JOBS.forEach(j => counts[j.name] = 0);
                    data.forEach(u => { if (u.jobName) counts[u.jobName] = (counts[u.jobName] || 0) + 1; });
                    setStats(counts);
                }, (err) => {
                    console.error("Snapshot error:", err);
                });
                return () => unsubscribe();
            }, [user, view]);

            const assignJobBalanced = async () => {
                const usersRef = collection(db, 'artifacts', appId, 'public', 'data', 'registrations');
                const allSnap = await getDocs(usersRef);
                const currentData = allSnap.docs.map(d => d.data());
                
                const counts = {};
                JOBS.forEach(j => counts[j.name] = 0);
                currentData.forEach(u => { if (u.jobName) counts[u.jobName] = (counts[u.jobName] || 0) + 1; });
                
                const minCount = Math.min(...Object.values(counts));
                const leastJobs = JOBS.filter(j => counts[j.name] === minCount);
                return leastJobs[Math.floor(Math.random() * leastJobs.length)];
            };

            const handleJoin = async (e) => {
                e.preventDefault();
                if (!user) { setError('서버 연결 중입니다. 잠시 후 시도해주세요.'); return; }
                if (!nickname.trim() || !password.trim()) { setError('닉네임과 비밀번호를 모두 입력해주세요.'); return; }
                setError('');
                setLoading(true);

                try {
                    const docRef = doc(db, 'artifacts', appId, 'public', 'data', 'registrations', nickname);
                    const docSnap = await getDoc(docRef);

                    if (docSnap.exists()) {
                        const data = docSnap.data();
                        if (data.password === password) {
                            setAssignment(data);
                            setView('result');
                        } else {
                            setError('비밀번호가 일치하지 않습니다.');
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
                    console.error("Join error:", err);
                    setError('데이터 저장 중 오류가 발생했습니다.');
                } finally {
                    setLoading(false);
                }
            };

            const handleAdminLogin = (e) => {
                e.preventDefault();
                if (adminInput === ADMIN_PASSWORD) {
                    setView('admin');
                    setError('');
                } else {
                    setError('관리자 비밀번호가 틀렸습니다.');
                }
            };

            if (loading && !user) {
                return (
                    <div className="min-h-screen flex flex-col items-center justify-center bg-slate-950">
                        <div className="animate-spin rounded-full h-12 w-12 border-t-2 border-b-2 border-indigo-500 mb-4"></div>
                        <p className="text-slate-400">모험가 길드 접속 중...</p>
                    </div>
                );
            }

            return (
                <div className="max-w-md mx-auto p-4 sm:p-8 min-h-screen flex flex-col">
                    <header className="text-center mb-10 pt-4 fade-in">
                        <h1 className="text-4xl font-black bg-gradient-to-r from-indigo-400 via-purple-400 to-pink-500 bg-clip-text text-transparent tracking-tighter">QUEST MASTER</h1>
                        <p className="text-slate-500 mt-2 font-medium tracking-wide">RPG 직업 자동 배정 시스템</p>
                    </header>

                    {view === 'login' && (
                        <div className="bg-slate-900 border border-white/10 rounded-3xl p-8 shadow-2xl fade-in">
                            <h2 className="text-xl font-bold mb-6 flex items-center gap-2">
                                <Icon name="users" className="w-5 h-5 text-indigo-400" /> 모험가 등록
                            </h2>
                            <form onSubmit={handleJoin} className="space-y-4">
                                <div className="space-y-1">
                                    <label className="text-[10px] font-bold text-slate-500 ml-1 uppercase">Nickname</label>
                                    <input type="text" value={nickname} onChange={e => setNickname(e.target.value)} className="w-full bg-slate-800 border border-slate-700 rounded-2xl px-5 py-4 outline-none focus:ring-2 focus:ring-indigo-500 transition-all" placeholder="닉네임(본명 추천)" />
                                </div>
                                <div className="space-y-1">
                                    <label className="text-[10px] font-bold text-slate-500 ml-1 uppercase">Password</label>
                                    <input type="password" value={password} onChange={e => setPassword(e.target.value)} className="w-full bg-slate-800 border border-slate-700 rounded-2xl px-5 py-4 outline-none focus:ring-2 focus:ring-indigo-500 transition-all" placeholder="본인 확인용 비밀번호" />
                                </div>
                                {error && <p className="text-red-400 text-sm bg-red-400/10 p-3 rounded-xl border border-red-400/20">{error}</p>}
                                <button type="submit" disabled={loading} className="w-full bg-indigo-600 hover:bg-indigo-500 font-black py-4 rounded-2xl transition-all shadow-lg shadow-indigo-500/20 active:scale-[0.98]">
                                    {loading ? '운명 결정 중...' : '나의 직업 확인'}
                                </button>
                            </form>
                            <button onClick={() => { setView('admin_auth'); setError(''); }} className="mt-8 w-full text-slate-600 hover:text-slate-400 text-[10px] flex items-center justify-center gap-1 font-bold tracking-widest transition-colors">
                                <Icon name="lock" className="w-3 h-3" /> ADMIN ACCESS
                            </button>
                        </div>
                    )}

                    {view === 'admin_auth' && (
                        <div className="bg-slate-900 border border-white/5 rounded-3xl p-8 shadow-2xl text-center fade-in">
                            <h2 className="text-xl font-bold text-yellow-500 mb-6 flex items-center justify-center gap-2">
                                <Icon name="key" className="w-5 h-5" /> 관리자 인증
                            </h2>
                            <form onSubmit={handleAdminLogin} className="space-y-4">
                                <input type="password" autoFocus value={adminInput} onChange={e => setAdminInput(e.target.value)} className="w-full bg-slate-800 border border-slate-700 rounded-2xl px-5 py-4 outline-none focus:ring-2 focus:ring-yellow-500" placeholder="관리자 암호" />
                                {error && <p className="text-red-400 text-sm mb-4">{error}</p>}
                                <div className="flex gap-2">
                                    <button type="button" onClick={() => setView('login')} className="flex-1 bg-slate-800 hover:bg-slate-700 py-4 rounded-2xl font-bold transition-all">취소</button>
                                    <button type="submit" className="flex-1 bg-yellow-600 hover:bg-yellow-500 py-4 rounded-2xl font-bold transition-all">확인</button>
                                </div>
                            </form>
                        </div>
                    )}

                    {view === 'result' && assignment && (
                        <div className="bg-slate-900 border border-white/10 rounded-[2.5rem] overflow-hidden shadow-2xl fade-in">
                            <div className={`${JOBS.find(j => j.id === assignment.jobId)?.color} p-12 text-center relative`}>
                                <div className="bg-white/20 w-24 h-24 rounded-full flex items-center justify-center mx-auto mb-4 backdrop-blur-md border border-white/30">
                                    <Icon name={JOBS.find(j => j.id === assignment.jobId)?.icon} className="w-12 h-12 text-white" />
                                </div>
                                <h2 className="text-5xl font-black text-white tracking-tighter drop-shadow-lg">{assignment.jobName}</h2>
                                <p className="text-white/80 mt-2 font-bold tracking-wide">{assignment.nickname} 용사님</p>
                            </div>
                            <div className="p-10 text-center bg-slate-900">
                                <p className="text-slate-300 italic mb-10 leading-relaxed font-medium">"{JOBS.find(j => j.id === assignment.jobId)?.desc}"</p>
                                <button onClick={() => { setView('login'); setNickname(''); setPassword(''); }} className="text-slate-500 hover:text-white text-sm font-bold flex items-center justify-center gap-2 mx-auto transition-colors">
                                    <Icon name="log-out" className="w-4 h-4" /> 로그아웃 후 처음으로
                                </button>
                            </div>
                        </div>
                    )}

                    {view === 'admin' && (
                        <div className="space-y-4 fade-in">
                            <div className="flex justify-between items-center bg-slate-900 p-5 rounded-2xl border border-white/5">
                                <h2 className="font-black flex items-center gap-3 text-lg">
                                    <Icon name="bar-chart-3" className="w-5 h-5 text-indigo-400" /> 길드 현황판
                                </h2>
                                <button onClick={() => setView('login')} className="text-[10px] font-bold bg-slate-800 hover:bg-slate-700 px-4 py-2 rounded-xl transition-all border border-slate-700">EXIT</button>
                            </div>
                            <div className="grid grid-cols-2 sm:grid-cols-3 gap-2">
                                <div className="bg-indigo-600 p-4 rounded-3xl text-center col-span-2 sm:col-span-1 shadow-lg shadow-indigo-600/20">
                                    <div className="text-3xl font-black">{allUsers.length}</div>
                                    <div className="text-[10px] font-bold opacity-70 uppercase tracking-widest">Total Users</div>
                                </div>
                                {JOBS.map(job => (
                                    <div key={job.id} className="bg-slate-900 border border-white/5 p-4 rounded-3xl text-center">
                                        <div className="text-2xl font-black">{stats[job.name] || 0}</div>
                                        <div className="text-[10px] font-bold text-slate-500 uppercase tracking-wider">{job.name}</div>
                                    </div>
                                ))}
                            </div>
                            <div className="bg-slate-900 border border-white/5 rounded-[2rem] overflow-hidden shadow-xl">
                                <div className="p-4 border-b border-white/5 bg-slate-800/30 flex justify-between items-center px-6">
                                    <span className="text-[10px] font-black text-slate-500 uppercase tracking-widest">Adventurer List</span>
                                    <span className="text-[10px] text-slate-600">최신순</span>
                                </div>
                                <div className="max-h-[350px] overflow-y-auto scrollbar-hide">
                                    <table className="w-full text-sm">
                                        <thead className="bg-slate-900/50 text-slate-500 text-[10px] font-black uppercase sticky top-0">
                                            <tr>
                                                <th className="px-6 py-4 text-left border-b border-white/5">Name</th>
                                                <th className="px-6 py-4 text-right border-b border-white/5">Class</th>
                                            </tr>
                                        </thead>
                                        <tbody className="divide-y divide-white/5">
                                            {allUsers.length === 0 ? (
                                                <tr><td colSpan="2" className="p-12 text-center text-slate-600 italic">아직 등록된 모험가가 없습니다.</td></tr>
                                            ) : (
                                                allUsers.map((u, i) => (
                                                    <tr key={i} className="hover:bg-white/5 transition-colors group">
                                                        <td className="px-6 py-4 font-bold text-slate-300 group-hover:text-white">{u.nickname}</td>
                                                        <td className="px-6 py-4 text-right">
                                                            <span className={`px-3 py-1 rounded-full text-[10px] font-black text-white ${JOBS.find(j => j.id === u.jobId)?.color}`}>
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
                    
                    <footer className="mt-auto pt-10 text-center text-slate-700 text-[9px] font-bold tracking-[0.3em] pb-6 uppercase">
                        &mdash; Guild Master Management System &mdash;
                    </footer>
                </div>
            );
        }

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);
    </script>
</body>
</html>
