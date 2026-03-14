<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>QUEST MASTER - 직업 배정 시스템</title>
    <!-- Tailwind CSS CDN -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- React & Babel CDN -->
    <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <!-- Lucide Icons -->
    <script src="https://unpkg.com/lucide@latest"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Pretendard:wght@400;700;900&display=swap');
        body { font-family: 'Pretendard', sans-serif; overflow-x: hidden; }
        .scrollbar-hide::-webkit-scrollbar { display: none; }
        .fade-in { animation: fadeIn 0.6s ease-out forwards; }
        @keyframes fadeIn {
            from { opacity: 0; transform: translateY(15px); }
            to { opacity: 1; transform: translateY(0); }
        }
    </style>
</head>
<body class="bg-slate-950 text-slate-100">
    <div id="root"></div>

    <script type="text/babel">
        const { useState, useEffect, useRef } = React;

        const STORAGE_KEY = 'rpg_event_registrations';
        const ADMIN_PASSWORD = "admin777"; 

        const JOBS = [
            { id: 'warrior', name: '전사', color: 'bg-red-500', icon: 'shield', desc: '강력한 체력으로 동료를 지키는 방패' },
            { id: 'mage', name: '마법사', color: 'bg-blue-500', icon: 'wand-2', desc: '강력한 원소 마법을 구사하는 현자' },
            { id: 'rogue', name: '도적', color: 'bg-purple-600', icon: 'sword', desc: '그림자 속에서 적을 기습하는 암살자' },
            { id: 'archer', name: '궁수', color: 'bg-green-600', icon: 'target', desc: '백발백중의 실력을 자랑하는 명사수' },
            { id: 'priest', name: '사제', color: 'bg-yellow-500', icon: 'cross', desc: '신성한 힘으로 상처를 치유하는 인도자' }
        ];

        const Icon = ({ name, className }) => {
            useEffect(() => { if (window.lucide) window.lucide.createIcons(); }, [name]);
            return <i data-lucide={name} className={className}></i>;
        };

        function App() {
            const [nickname, setNickname] = useState('');
            const [password, setPassword] = useState('');
            const [loading, setLoading] = useState(true);
            const [assignment, setAssignment] = useState(null);
            const [error, setError] = useState('');
            const [view, setView] = useState('login'); 
            const [adminInput, setAdminInput] = useState('');
            const [allUsers, setAllUsers] = useState([]);
            const [stats, setStats] = useState({});

            // 데이터 로드 및 초기화
            useEffect(() => {
                const timer = setTimeout(() => {
                    loadData();
                    setLoading(false);
                }, 800);
                return () => clearTimeout(timer);
            }, []);

            const loadData = () => {
                const saved = localStorage.getItem(STORAGE_KEY);
                if (saved) {
                    const data = JSON.parse(saved);
                    setAllUsers(data);
                    calculateStats(data);
                }
            };

            const calculateStats = (data) => {
                const counts = {};
                JOBS.forEach(j => counts[j.name] = 0);
                data.forEach(u => { if (u.jobName) counts[u.jobName] = (counts[u.jobName] || 0) + 1; });
                setStats(counts);
            };

            const assignJobBalanced = (currentData) => {
                const counts = {};
                JOBS.forEach(j => counts[j.name] = 0);
                currentData.forEach(u => { if (u.jobName) counts[u.jobName]++; });

                const min = Math.min(...Object.values(counts));
                const candidates = JOBS.filter(j => counts[j.name] === min);
                return candidates[Math.floor(Math.random() * candidates.length)];
            };

            const handleJoin = (e) => {
                e.preventDefault();
                if (!nickname.trim() || !password.trim()) { setError('모든 항목을 입력해주세요.'); return; }
                setError('');
                setLoading(true);

                // 로컬 스토리지 데이터 처리
                const currentData = JSON.parse(localStorage.getItem(STORAGE_KEY) || '[]');
                const existingUser = currentData.find(u => u.nickname === nickname);

                if (existingUser) {
                    if (existingUser.password === password) {
                        setAssignment(existingUser);
                        setView('result');
                    } else {
                        setError('비밀번호가 일치하지 않습니다.');
                    }
                } else {
                    const selected = assignJobBalanced(currentData);
                    const newEntry = { 
                        nickname, 
                        password, 
                        jobId: selected.id, 
                        jobName: selected.name, 
                        timestamp: Date.now() 
                    };
                    
                    const updatedData = [...currentData, newEntry];
                    localStorage.setItem(STORAGE_KEY, JSON.stringify(updatedData));
                    setAllUsers(updatedData);
                    calculateStats(updatedData);
                    setAssignment(newEntry);
                    setView('result');
                }
                setLoading(false);
            };

            const handleAdminLogin = (e) => {
                e.preventDefault();
                if (adminInput === ADMIN_PASSWORD) {
                    loadData();
                    setView('admin');
                    setError('');
                } else {
                    setError('관리자 비밀번호가 틀렸습니다.');
                }
            };

            if (loading) return (
                <div className="min-h-screen flex flex-col items-center justify-center bg-slate-950 text-indigo-400">
                    <div className="animate-spin rounded-full h-10 w-10 border-t-2 border-indigo-500 mb-4"></div>
                    <p className="text-sm font-bold tracking-widest animate-pulse">LOADING REALM...</p>
                </div>
            );

            return (
                <div className="max-w-md mx-auto p-4 sm:p-8 min-h-screen flex flex-col">
                    <header className="text-center mb-10 pt-4 fade-in">
                        <h1 className="text-4xl font-black bg-gradient-to-r from-indigo-400 to-pink-500 bg-clip-text text-transparent">QUEST MASTER</h1>
                        <p className="text-slate-500 mt-2 font-medium">RPG 직업 배정 시스템</p>
                    </header>

                    {view === 'login' && (
                        <div className="bg-slate-900 border border-white/10 rounded-3xl p-8 shadow-2xl fade-in">
                            <h2 className="text-xl font-bold mb-6 flex items-center gap-2"><Icon name="users" className="w-5 h-5 text-indigo-400" /> 모험가 등록</h2>
                            <form onSubmit={handleJoin} className="space-y-4">
                                <input type="text" value={nickname} onChange={e => setNickname(e.target.value)} className="w-full bg-slate-800 border border-slate-700 rounded-2xl px-5 py-4 outline-none focus:ring-2 focus:ring-indigo-500 transition-all" placeholder="닉네임(본명)" />
                                <input type="password" value={password} onChange={e => setPassword(e.target.value)} className="w-full bg-slate-800 border border-slate-700 rounded-2xl px-5 py-4 outline-none focus:ring-2 focus:ring-indigo-500 transition-all" placeholder="비밀번호" />
                                {error && <p className="text-red-400 text-sm bg-red-400/5 p-3 rounded-xl border border-red-400/10">{error}</p>}
                                <button type="submit" className="w-full bg-indigo-600 hover:bg-indigo-500 font-black py-4 rounded-2xl shadow-lg transition-all active:scale-95">운명 결정하기</button>
                            </form>
                            <button onClick={() => { setView('admin_auth'); setError(''); }} className="mt-8 w-full text-slate-600 text-[10px] font-bold tracking-widest uppercase hover:text-slate-400 transition-colors">Admin Login</button>
                        </div>
                    )}

                    {view === 'admin_auth' && (
                        <div className="bg-slate-900 border border-white/5 rounded-3xl p-8 fade-in">
                            <h2 className="text-xl font-bold text-yellow-500 mb-6 flex items-center justify-center gap-2"><Icon name="key" className="w-5 h-5" /> 관리자 인증</h2>
                            <form onSubmit={handleAdminLogin} className="space-y-4">
                                <input type="password" autoFocus value={adminInput} onChange={e => setAdminInput(e.target.value)} className="w-full bg-slate-800 border border-slate-700 rounded-2xl px-5 py-4 mb-4 outline-none focus:ring-2 focus:ring-yellow-500" placeholder="비밀번호 입력" />
                                {error && <p className="text-red-400 text-sm mb-4">{error}</p>}
                                <div className="flex gap-2">
                                    <button type="button" onClick={() => setView('login')} className="flex-1 bg-slate-800 py-4 rounded-2xl font-bold">취소</button>
                                    <button type="submit" className="flex-1 bg-yellow-600 py-4 rounded-2xl font-bold">확인</button>
                                </div>
                            </form>
                        </div>
                    )}

                    {view === 'result' && assignment && (
                        <div className="bg-slate-900 border border-white/10 rounded-[2.5rem] overflow-hidden shadow-2xl fade-in">
                            <div className={`${JOBS.find(j => j.id === assignment.jobId)?.color} p-12 text-center`}>
                                <div className="bg-white/20 w-24 h-24 rounded-full flex items-center justify-center mx-auto mb-4 border border-white/30 backdrop-blur-md shadow-inner">
                                    <Icon name={JOBS.find(j => j.id === assignment.jobId)?.icon} className="w-12 h-12 text-white" />
                                </div>
                                <h2 className="text-5xl font-black text-white tracking-tighter drop-shadow-lg">{assignment.jobName}</h2>
                                <p className="text-white/80 mt-2 font-bold">{assignment.nickname} 용사님</p>
                            </div>
                            <div className="p-10 text-center">
                                <p className="text-slate-300 italic mb-8 leading-relaxed font-medium">"{JOBS.find(j => j.id === assignment.jobId)?.desc}"</p>
                                <button onClick={() => { setView('login'); setNickname(''); setPassword(''); }} className="text-slate-500 font-bold flex items-center justify-center gap-2 mx-auto hover:text-white transition-colors"><Icon name="log-out" className="w-4 h-4" /> 로그아웃</button>
                            </div>
                        </div>
                    )}

                    {view === 'admin' && (
                        <div className="space-y-4 fade-in">
                            <div className="flex justify-between items-center bg-slate-900 p-5 rounded-2xl border border-white/5">
                                <h2 className="font-black flex items-center gap-2"><Icon name="bar-chart-3" className="w-5 h-5 text-indigo-400" /> 실시간 현황</h2>
                                <button onClick={() => setView('login')} className="text-xs bg-slate-800 px-4 py-2 rounded-xl border border-slate-700">종료</button>
                            </div>
                            <div className="grid grid-cols-2 gap-2">
                                <div className="bg-indigo-600 p-4 rounded-3xl text-center col-span-2 shadow-lg">
                                    <div className="text-3xl font-black">{allUsers.length}</div>
                                    <div className="text-[10px] font-bold opacity-70 uppercase tracking-widest">Total Adventurers</div>
                                </div>
                                {JOBS.map(job => (
                                    <div key={job.id} className="bg-slate-900 border border-white/5 p-4 rounded-3xl text-center">
                                        <div className="text-xl font-black">{stats[job.name] || 0}</div>
                                        <div className="text-[10px] text-slate-500 uppercase font-bold">{job.name}</div>
                                    </div>
                                ))}
                            </div>
                            <div className="bg-slate-900 border border-white/5 rounded-[2rem] overflow-hidden max-h-[350px] overflow-y-auto scrollbar-hide">
                                <table className="w-full text-sm">
                                    <thead className="bg-slate-800 text-slate-500 text-[10px] font-black uppercase sticky top-0 border-b border-white/5">
                                        <tr><th className="px-6 py-4 text-left">이름</th><th className="px-6 py-4 text-right">직업</th></tr>
                                    </thead>
                                    <tbody className="divide-y divide-white/5">
                                        {allUsers.length === 0 ? (
                                            <tr><td colSpan="2" className="p-12 text-center text-slate-600 italic">데이터가 없습니다.</td></tr>
                                        ) : (
                                            [...allUsers].sort((a,b) => b.timestamp - a.timestamp).map((u, i) => (
                                                <tr key={i} className="hover:bg-white/5 transition-colors">
                                                    <td className="px-6 py-4 font-bold">{u.nickname}</td>
                                                    <td className="px-6 py-4 text-right">
                                                        <span className={`px-2 py-0.5 rounded-full text-[10px] font-bold text-white shadow-sm ${JOBS.find(j => j.id === u.jobId)?.color}`}>
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
                    )}
                </div>
            );
        }

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);
    </script>
</body>
</html>
