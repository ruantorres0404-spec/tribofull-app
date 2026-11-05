# tribofull-app
import React, { useState, useEffect, useMemo, useRef, useCallback } from 'react';
import { Trophy, Lock, History, Plus, CheckCircle, Trash2, Award, LogOut, Save, X, Users, Timer, Play, Square, RotateCcw, AlertTriangle, Shield, Zap, ChevronRight, UserCheck, Edit, Info, Eye, EyeOff, Scroll, MessageSquare, Swords, Crosshair, Wand2, HeartPulse } from 'lucide-react';

// ==========================================
// CONFIGURAÇÃO DO FIREBASE (MANDATORY RULES)
// ==========================================
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, setDoc, deleteDoc, onSnapshot, collection } from 'firebase/firestore';

const hasFirebase = typeof __firebase_config !== 'undefined' && __firebase_config;
let app, auth, db, appId;

if (hasFirebase) {
  try {
    const firebaseConfig = JSON.parse(__firebase_config);
    app = initializeApp(firebaseConfig);
    auth = getAuth(app);
    db = getFirestore(app);
    appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
  } catch (error) {
    console.error("Erro ao inicializar o Firebase:", error);
  }
}

// ==========================================
// CONFIGURAÇÃO DAS TRIBOS / HUNTERS
// ==========================================
const DEFAULT_TEAM_CONFIG = [
  { 
    id: 'A', name: 'Garden', color: 'bg-yellow-500/80', text: 'text-yellow-400', 
    lightBg: 'bg-yellow-950/40', 
    fireGlow: '0 0 25px rgba(234, 179, 8, 0.5), inset 0 0 15px rgba(234, 179, 8, 0.2)',
    fireBorder: 'rgba(234, 179, 8, 0.7)',
    leader: 'Líder Garden',
    participants: [{ name: 'Membro 1', role: 'Caçador' }, { name: 'Membro 2', role: 'Caçador' }],
    description: 'Tribo amarela, focada em crescimento, adaptação e vitalidade inabalável.' 
  },
  { 
    id: 'B', name: 'Flock', color: 'bg-blue-600/80', text: 'text-blue-400', 
    lightBg: 'bg-blue-950/40', 
    fireGlow: '0 0 25px rgba(37, 99, 235, 0.5), inset 0 0 15px rgba(37, 99, 235, 0.2)',
    fireBorder: 'rgba(37, 99, 235, 0.7)',
    leader: 'Líder Flock',
    participants: [{ name: 'Membro 1', role: 'Caçador' }, { name: 'Membro 2', role: 'Caçador' }],
    description: 'Tribo azul, voando alto com visão estratégica e precisão absoluta.' 
  },
  { 
    id: 'C', name: 'Bold', color: 'bg-orange-500/80', text: 'text-orange-400', 
    lightBg: 'bg-orange-950/40', 
    fireGlow: '0 0 25px rgba(249, 115, 22, 0.5), inset 0 0 15px rgba(249, 115, 22, 0.2)',
    fireBorder: 'rgba(249, 115, 22, 0.7)',
    leader: 'Líder Bold',
    participants: [{ name: 'Membro 1', role: 'Caçador' }, { name: 'Membro 2', role: 'Caçador' }],
    description: 'Tribo laranja, destemida e corajosa, enfrentando todos os desafios de frente.' 
  },
  { 
    id: 'D', name: 'Burn', color: 'bg-red-600/80', text: 'text-red-400', 
    lightBg: 'bg-red-950/40', 
    fireGlow: '0 0 25px rgba(220, 38, 38, 0.5), inset 0 0 15px rgba(220, 38, 38, 0.2)',
    fireBorder: 'rgba(220, 38, 38, 0.7)',
    leader: 'Líder Burn',
    participants: [{ name: 'Membro 1', role: 'Caçador' }, { name: 'Membro 2', role: 'Caçador' }],
    description: 'Tribo vermelha, consumindo os obstáculos com a força e fúria das chamas.' 
  },
];

// ==========================================
// SUB-COMPONENTES OTIMIZADOS
// ==========================================

const ToastNotification = ({ toast }) => {
  if (!toast.show) return null;
  return (
    <div className="fixed bottom-6 right-6 z-50 animate-fade-in-up">
      <div className={`flex items-center gap-3 px-5 py-3 rounded shadow-2xl font-montserrat font-bold text-xs uppercase tracking-widest border backdrop-blur-md ${toast.type === 'success' ? 'bg-cyan-950/90 border-cyan-400 text-cyan-300 shadow-[0_0_15px_rgba(6,182,212,0.5)]' : 'bg-red-950/90 border-red-500 text-red-300 shadow-[0_0_15px_rgba(239,68,68,0.5)]'}`}>
        {toast.type === 'success' ? <CheckCircle className="w-4 h-4 text-cyan-400" /> : <AlertTriangle className="w-4 h-4 text-red-400" />}
        {toast.message}
      </div>
    </div>
  );
};

const QuestBoardTab = ({ pendingTasks, settings }) => {
  return (
    <div className="space-y-10 animate-fade-in relative z-10">
      <section className="solo-panel p-6 sm:p-10 rounded-xl relative border-l-4 border-l-cyan-400">
        <h2 className="text-sm font-montserrat font-bold text-cyan-300 mb-4 flex items-center gap-2 uppercase tracking-widest"><MessageSquare className="w-5 h-5 text-cyan-400" /> Aviso do Sistema</h2>
        <div className="p-4 bg-slate-900/60 rounded border border-cyan-500/20 shadow-[inset_0_0_15px_rgba(6,182,212,0.1)]">
           <p className="text-cyan-100 font-inter leading-relaxed whitespace-pre-wrap">{settings.systemMessage || 'Nenhum aviso do Monarca no momento.'}</p>
        </div>
      </section>
      
      <section>
        <h2 className="text-lg sm:text-xl font-montserrat font-bold text-cyan-300 uppercase tracking-widest mb-6 flex items-center gap-3"><Scroll className="w-5 h-5 text-cyan-400" /> [MASMORRAS PENDENTES]</h2>
        {pendingTasks.length === 0 ? (
          <div className="text-center py-16 bg-slate-900/60 rounded-lg border border-cyan-500/20">
            <Shield className="w-10 h-10 text-cyan-600/50 mx-auto mb-4 animate-pulse" />
            <p className="text-cyan-500/70 font-mono text-xs tracking-widest uppercase">Nenhuma masmorra detectada no radar.</p>
          </div>
        ) : (
          <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
            {pendingTasks.map(task => (
              <div key={task.id} className="solo-panel p-6 rounded-xl border border-red-500/30 shadow-[0_0_15px_rgba(239,68,68,0.1)] hover:border-red-400/50 transition-colors relative overflow-hidden group">
                <div className="absolute top-0 right-0 bg-red-950/80 text-red-400 text-[10px] font-mono font-bold px-3 py-1 border-b border-l border-red-500/30 uppercase tracking-widest">Ameaça Iminente</div>
                <div className="flex items-center gap-3 mb-2 mt-2">
                  <Swords className="w-5 h-5 text-red-400 drop-shadow-[0_0_8px_rgba(239,68,68,0.6)]" />
                  <h3 className="font-montserrat font-bold text-red-100 uppercase tracking-wide text-lg">{task.name}</h3>
                </div>
                <p className="text-slate-400 font-mono text-xs mt-4 group-hover:text-cyan-300 transition-colors">❯❯ STATUS: AGUARDANDO CAÇADORES...</p>
              </div>
            ))}
          </div>
        )}
      </section>
    </div>
  );
};

const RankingTab = ({ teamScores, completedTasks, teams, currentUser, settings }) => {
  const isHidden = !settings.isRankingVisible && currentUser.role !== 'admin';

  const podiumOrder = [
    { rank: 2, team: teamScores[1] || teamScores[0], height: 'h-32 sm:h-48', iconColor: 'text-slate-300' },
    { rank: 1, team: teamScores[0], height: 'h-40 sm:h-64', iconColor: 'text-cyan-400' },
    { rank: 3, team: teamScores[2] || teamScores[0], height: 'h-24 sm:h-36', iconColor: 'text-teal-600' },
    { rank: 4, team: teamScores[3] || teamScores[0], height: 'h-16 sm:h-24', iconColor: 'text-slate-500' },
  ];

  return (
    <div className="space-y-12 animate-fade-in relative z-10">
      <section className="solo-panel p-6 sm:p-10 rounded-xl relative">
        <div className="absolute top-0 left-0 w-3 h-3 border-t-2 border-l-2 border-cyan-400"></div>
        <div className="absolute top-0 right-0 w-3 h-3 border-t-2 border-r-2 border-cyan-400"></div>
        <div className="absolute bottom-0 left-0 w-3 h-3 border-b-2 border-l-2 border-cyan-400"></div>
        <div className="absolute bottom-0 right-0 w-3 h-3 border-b-2 border-r-2 border-cyan-400"></div>

        <h2 className="text-xl sm:text-2xl font-montserrat font-bold text-center text-cyan-300 mb-2 uppercase tracking-[0.25em] drop-shadow-[0_0_10px_rgba(6,182,212,0.5)]">[RANKING DE TRIBOS]</h2>
        <p className="text-center text-cyan-600/80 font-mono text-xs tracking-widest mb-12">❯❯ STATUS DAS GUILDAS EM TEMPO REAL ❮❮</p>
        
        {isHidden ? (
          <div className="text-center py-20 bg-slate-950/80 rounded-lg border border-cyan-500/20 relative overflow-hidden">
            <div className="absolute inset-0 bg-[repeating-linear-gradient(45deg,transparent,transparent_10px,rgba(6,182,212,0.05)_10px,rgba(6,182,212,0.05)_20px)]"></div>
            <Lock className="w-12 h-12 text-cyan-600/50 mx-auto mb-6 animate-pulse relative z-10" />
            <p className="text-cyan-400 font-montserrat font-bold text-lg tracking-widest uppercase relative z-10 drop-shadow-[0_0_8px_rgba(6,182,212,0.5)]">[CLASSIFICAÇÃO RESTRITA PELO SISTEMA]</p>
            <p className="text-cyan-600/70 font-mono text-xs mt-3 uppercase tracking-widest relative z-10">O Monarca ainda não revelou os resultados.</p>
          </div>
        ) : completedTasks.length !== 0 ? (
          <div className="flex justify-center items-end gap-2 sm:gap-6 mt-12 mb-4 px-2">
            {podiumOrder.map((pos, index) => (
              <div key={index} className="flex flex-col items-center w-20 sm:w-32">
                <div className="flex flex-col items-center mb-4">
                  <Award className={`w-6 h-6 sm:w-8 sm:h-8 mb-2 ${pos.iconColor} drop-shadow-[0_0_8px_rgba(6,182,212,0.8)]`} />
                  <span className={`font-montserrat font-bold text-[10px] sm:text-xs text-center ${pos.team.text} uppercase tracking-wider whitespace-nowrap`}>{pos.team.name}</span>
                  <span className="font-mono font-bold text-cyan-200 text-sm sm:text-lg mt-1 drop-shadow-[0_0_5px_rgba(6,182,212,0.5)]">{pos.team.points} XP</span>
                </div>
                <div className={`w-full ${pos.team.color} ${pos.height} rounded-t-sm shadow-[0_0_15px_rgba(6,182,212,0.3)] border-t border-l border-r border-cyan-400/40 flex justify-center pt-3 sm:pt-5 transition-all duration-500 relative overflow-hidden`}>
                  <div className="absolute inset-0 bg-gradient-to-b from-cyan-400/20 to-transparent"></div>
                  <span className="text-white font-montserrat font-bold text-xl sm:text-2xl opacity-90 relative z-10">{pos.rank}</span>
                </div>
              </div>
            ))}
          </div>
        ) : (
          <div className="text-center py-16 bg-slate-900/60 rounded-lg border border-cyan-500/20">
            <Shield className="w-10 h-10 text-cyan-600/50 mx-auto mb-4 animate-pulse" />
            <p className="text-cyan-500/70 font-mono text-xs tracking-widest uppercase">[NENHUMA MASMORRA CONCLUÍDA AINDA]</p>
          </div>
        )}
      </section>

      <section>
        <div className="flex items-center gap-3 mb-6">
          <History className="w-5 h-5 text-cyan-400" />
          <h2 className="text-lg sm:text-xl font-montserrat font-bold text-cyan-300 uppercase tracking-widest">[HISTÓRICO DE PROVAS]</h2>
        </div>
        {completedTasks.length === 0 ? (
          <p className="text-cyan-600/70 text-center py-8 solo-panel rounded-lg font-mono text-xs uppercase tracking-widest">Nenhum registro de combate encontrado.</p>
        ) : (
          <div className="space-y-4">
            {completedTasks.map(task => (
              <div key={task.id} className="solo-panel rounded-xl overflow-hidden transition-all hover:border-cyan-500/50">
                <div className="bg-slate-900/80 px-5 py-4 border-b border-cyan-500/20 flex items-center justify-between">
                  <h3 className="font-montserrat font-bold text-sm sm:text-base text-cyan-200 uppercase tracking-wide">{task.name}</h3>
                  <div className="flex items-center gap-2 text-cyan-400 bg-cyan-950/60 px-3 py-1 rounded text-xs font-mono font-bold tracking-widest border border-cyan-500/40 shadow-[0_0_8px_rgba(6,182,212,0.3)]">
                    <CheckCircle className="w-3.5 h-3.5" /> LIMPA
                  </div>
                </div>
                <div className="p-5 grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-4 bg-slate-950/50">
                  {task.results && [...task.results].sort((a,b) => a.place - b.place).map(result => {
                    const team = teams.find(t => t.id === result.teamId);
                    if (!team) return null;
                    const showPoints = !isHidden || (currentUser.role === 'hunter' && currentUser.tribeId === team.id);

                    return (
                      <div key={result.place} className={`flex items-center gap-4 p-3 rounded-lg border-l-2 ${team.lightBg} border-l-cyan-400 border border-cyan-900/40 opacity-90`}>
                        <div className="font-montserrat font-bold text-xl text-cyan-600 w-6">#{result.place}</div>
                        <div>
                          <div className={`font-inter font-bold text-xs uppercase tracking-wider ${team.text}`}>{team.name}</div>
                          <div className="text-cyan-200 font-mono text-sm">{showPoints ? `${result.points} XP` : '??? XP'}</div>
                        </div>
                      </div>
                    );
                  })}
                </div>
              </div>
            ))}
          </div>
        )}
      </section>
    </div>
  );
};

const TribosTab = ({ teamScores, allTasks, onUpdateTeam, currentUser, settings }) => {
  const [selectedTeamModal, setSelectedTeamModal] = useState(null);
  const [isEditing, setIsEditing] = useState(false);
  const [editForm, setEditForm] = useState(null);

  const isHidden = !settings.isRankingVisible && currentUser.role !== 'admin';

  const handleOpenModal = (team) => {
    setSelectedTeamModal(team);
    setEditForm(JSON.parse(JSON.stringify(team)));
    setIsEditing(false);
  };

  const handleParticipantChange = (index, field, value) => {
    const updated = [...editForm.participants];
    const current = typeof updated[index] === 'string' ? { name: updated[index], role: 'Caçador' } : updated[index];
    updated[index] = { ...current, [field]: value };
    setEditForm({ ...editForm, participants: updated });
  };

  const handleAddParticipant = () => {
    setEditForm({ ...editForm, participants: [...editForm.participants, { name: "", role: "Caçador" }] });
  };

  const handleRemoveParticipant = (index) => {
    const updated = editForm.participants.filter((_, i) => i !== index);
    setEditForm({ ...editForm, participants: updated });
  };

  const handleSaveTeam = () => {
    onUpdateTeam(editForm);
    setSelectedTeamModal(editForm);
    setIsEditing(false);
  };

  const canEditTeam = (teamId) => {
    return currentUser.role === 'admin' || (currentUser.role === 'leader' && currentUser.tribeId === teamId);
  };

  return (
    <div className="animate-fade-in space-y-8 relative z-10">
      <div className="text-center mb-10">
        <h2 className="text-xl sm:text-2xl font-montserrat font-bold text-cyan-300 uppercase tracking-[0.25em] mb-2 drop-shadow-[0_0_8px_rgba(6,182,212,0.5)]">[STATUS DE TRIBOS]</h2>
        <p className="text-cyan-600 font-mono text-xs tracking-widest">❯❯ CLIQUE EM UMA GUILDA PARA ACESSAR DETALHES ❮❮</p>
      </div>

      <div className="grid grid-cols-1 md:grid-cols-2 gap-8">
        {teamScores.map(team => (
          <div key={team.id} onClick={() => handleOpenModal(team)} className="solo-panel rounded-xl p-6 flex flex-col justify-between transition-all duration-300 hover:scale-[1.01] cursor-pointer group relative" style={{ boxShadow: team.fireGlow, borderColor: team.fireBorder }}>
            <div className="absolute top-2 right-2 w-2 h-2 border-t border-r border-cyan-400 group-hover:scale-125 transition-transform"></div>
            <div>
              <div className="flex items-center justify-between mb-5">
                <div className="flex items-center gap-4">
                  <div className={`w-10 h-10 rounded bg-slate-900 border border-cyan-500/40 flex items-center justify-center shadow-[0_0_10px_rgba(6,182,212,0.3)]`}>
                    <Users className={`w-5 h-5 ${team.text}`} />
                  </div>
                  <h3 className={`text-lg font-montserrat font-bold uppercase tracking-widest ${team.text}`}>{team.name}</h3>
                </div>
                <ChevronRight className="w-5 h-5 text-cyan-500 group-hover:translate-x-1 transition-transform" />
              </div>
              <p className="text-cyan-600/90 mb-8 font-inter text-sm font-light leading-relaxed border-l border-cyan-500/30 pl-4">{team.description}</p>
            </div>
            <div className="grid grid-cols-2 gap-4 bg-slate-950 p-4 rounded-lg border border-cyan-900/60 relative overflow-hidden">
              {isHidden && <div className="absolute inset-0 bg-slate-950/80 backdrop-blur-[2px] z-10 flex items-center justify-center border border-cyan-900/50"><Lock className="w-4 h-4 text-cyan-600 mr-2"/><span className="text-cyan-500 font-mono text-xs uppercase">Restrito</span></div>}
              <div className="text-center">
                <div className="text-[10px] font-montserrat font-bold text-cyan-600 uppercase tracking-widest mb-1">XP Total</div>
                <div className="text-2xl font-mono font-bold text-cyan-200 drop-shadow-[0_0_5px_rgba(6,182,212,0.4)]">{isHidden ? '???' : team.points}</div>
              </div>
              <div className="text-center border-l border-cyan-900/60">
                <div className="text-[10px] font-montserrat font-bold text-cyan-600 uppercase tracking-widest mb-1">Média / Prova</div>
                <div className="text-2xl font-mono font-bold text-cyan-400">{isHidden ? '???' : team.average}</div>
              </div>
            </div>
          </div>
        ))}
      </div>

      {selectedTeamModal && (
        <div className="fixed inset-0 bg-slate-950/85 backdrop-blur-sm z-50 flex items-center justify-center p-4 animate-fade-in">
          <div className="solo-panel rounded-xl p-6 sm:p-8 max-w-lg w-full shadow-2xl animate-fade-in-up border relative max-h-[90vh] overflow-y-auto" style={{ boxShadow: selectedTeamModal.fireGlow, borderColor: selectedTeamModal.fireBorder }}>
            <div className="absolute top-0 left-0 w-2.5 h-2.5 border-t-2 border-l-2 border-cyan-400"></div>
            <div className="absolute top-0 right-0 w-2.5 h-2.5 border-t-2 border-r-2 border-cyan-400"></div>
            <div className="flex justify-between items-start mb-6">
              <div className="flex-1 mr-4">
                <div className="text-xs font-montserrat font-bold text-cyan-600 uppercase tracking-widest">[DOSSIÊ DA GUILDA]</div>
                {!isEditing ? (
                  <h3 className={`text-xl font-montserrat font-bold uppercase tracking-wider ${selectedTeamModal.text} mt-1 break-words`}>{selectedTeamModal.name}</h3>
                ) : (
                  <input type="text" value={editForm.name} onChange={(e) => setEditForm({ ...editForm, name: e.target.value })} className="mt-1 w-full px-3 py-1.5 rounded bg-slate-900 border border-cyan-500/50 text-cyan-200 outline-none font-montserrat font-bold text-lg uppercase tracking-wider" placeholder="Nome da Tribo" />
                )}
              </div>
              <div className="flex items-center gap-2">
                {!isEditing && canEditTeam(selectedTeamModal.id) && (
                  <button onClick={() => setIsEditing(true)} className="text-cyan-500 hover:text-cyan-300 p-1.5 hover:bg-cyan-950/50 rounded transition-colors"><Edit className="w-5 h-5" /></button>
                )}
                <button onClick={() => { setSelectedTeamModal(null); setIsEditing(false); }} className="text-cyan-500 hover:text-cyan-300 p-1.5 hover:bg-cyan-950/50 rounded transition-colors"><X className="w-6 h-6" /></button>
              </div>
            </div>
            <div className="space-y-6">
              <div className="bg-slate-900/80 p-4 rounded-lg border border-cyan-500/30">
                <div className="flex items-center gap-2 text-xs font-montserrat font-bold text-cyan-400 uppercase tracking-widest mb-2"><UserCheck className="w-4 h-4" /> Líder da Tribo</div>
                {!isEditing ? (
                  <div className="text-cyan-100 font-inter font-medium text-sm pl-6">{selectedTeamModal.leader}</div>
                ) : (
                  <div className="pl-6">
                    <input type="text" value={editForm.leader} onChange={(e) => setEditForm({ ...editForm, leader: e.target.value })} className="w-full px-3 py-2 rounded bg-slate-950 border border-cyan-500/50 text-cyan-200 outline-none font-inter text-sm" placeholder="Nome do Líder" />
                  </div>
                )}
              </div>

              <div className="bg-slate-900/80 p-4 rounded-lg border border-cyan-500/30">
                <div className="flex items-center gap-2 text-xs font-montserrat font-bold text-cyan-400 uppercase tracking-widest mb-2"><Info className="w-4 h-4" /> Descrição</div>
                {!isEditing ? (
                  <div className="text-cyan-100 font-inter font-medium text-sm pl-6">{selectedTeamModal.description}</div>
                ) : (
                  <div className="pl-6">
                    <textarea value={editForm.description} onChange={(e) => setEditForm({ ...editForm, description: e.target.value })} className="w-full px-3 py-2 rounded bg-slate-950 border border-cyan-500/50 text-cyan-200 outline-none font-inter text-sm resize-none h-20" placeholder="Descrição da Tribo..." />
                  </div>
                )}
              </div>

              <div className="bg-slate-900/80 p-4 rounded-lg border border-cyan-500/30">
                <div className="flex items-center justify-between text-xs font-montserrat font-bold text-cyan-400 uppercase tracking-widest mb-3">
                  <span className="flex items-center gap-2"><Users className="w-4 h-4" /> Caçadores {!isEditing && `(${selectedTeamModal.participants.length})`}</span>
                </div>
                {!isEditing ? (
                  <div className="space-y-2 pl-4">
                    {selectedTeamModal.participants.map((m, idx) => {
                      const pData = typeof m === 'string' ? { name: m, role: 'Caçador' } : m;
                      const RoleIcon = pData.role === 'Tanque' ? Shield : pData.role === 'Mago' ? Wand2 : pData.role === 'Assassino' ? Swords : pData.role === 'Curandeiro' ? HeartPulse : Crosshair;
                      const roleColor = pData.role === 'Tanque' ? 'text-amber-400' : pData.role === 'Mago' ? 'text-purple-400' : pData.role === 'Assassino' ? 'text-red-400' : pData.role === 'Curandeiro' ? 'text-emerald-400' : 'text-cyan-400';
                      return (
                        <div key={idx} className="flex justify-between items-center bg-slate-950/60 p-2.5 rounded border border-cyan-900/30">
                          <div className="text-xs font-inter text-cyan-100 flex items-center gap-2"><span className="w-1.5 h-1.5 bg-cyan-400 rounded-full"></span> {pData.name}</div>
                          <div className={`flex items-center gap-1.5 text-[10px] font-mono font-bold uppercase tracking-widest ${roleColor}`}><RoleIcon className="w-3.5 h-3.5" /> {pData.role}</div>
                        </div>
                      );
                    })}
                  </div>
                ) : (
                  <div className="space-y-2 pl-4">
                    {editForm.participants.map((m, idx) => {
                      const pData = typeof m === 'string' ? { name: m, role: 'Caçador' } : m;
                      return (
                        <div key={idx} className="flex flex-col sm:flex-row items-start sm:items-center gap-2">
                          <input type="text" value={pData.name} onChange={(e) => handleParticipantChange(idx, 'name', e.target.value)} className="flex-1 w-full px-3 py-1.5 rounded bg-slate-950 border border-cyan-500/50 text-cyan-200 outline-none font-inter text-xs" placeholder={`Caçador ${idx + 1}`} />
                          <div className="flex w-full sm:w-auto items-center gap-2">
                            <select value={pData.role} onChange={(e) => handleParticipantChange(idx, 'role', e.target.value)} className="flex-1 sm:w-28 px-2 py-1.5 rounded bg-slate-950 border border-cyan-500/50 text-cyan-400 outline-none font-mono text-[10px] uppercase">
                              <option value="Caçador">Caçador</option>
                              <option value="Tanque">Tanque</option>
                              <option value="Mago">Mago</option>
                              <option value="Assassino">Assassino</option>
                              <option value="Curandeiro">Curand.</option>
                            </select>
                            <button onClick={() => handleRemoveParticipant(idx)} className="text-red-400 hover:text-red-300 p-1.5 bg-red-950/30 rounded border border-red-900/50"><Trash2 className="w-4 h-4" /></button>
                          </div>
                        </div>
                      );
                    })}
                    <button onClick={handleAddParticipant} className="mt-3 text-xs font-montserrat font-bold text-cyan-400 flex items-center gap-1 hover:text-cyan-200 transition-colors uppercase"><Plus className="w-3.5 h-3.5" /> Recrutar</button>
                  </div>
                )}
              </div>
              
              {!isEditing && (
                <div className="bg-slate-900/80 p-4 rounded-lg border border-cyan-500/30">
                  <div className="flex items-center gap-2 text-xs font-montserrat font-bold text-cyan-400 uppercase tracking-widest mb-3"><History className="w-4 h-4" /> Provas e Colocações</div>
                  <div className="space-y-2 max-h-48 overflow-y-auto pr-1">
                    {allTasks.filter(t => t.status === 'completed').length === 0 ? (
                      <p className="text-xs text-cyan-600 font-mono italic pl-6">Nenhuma prova concluída ainda.</p>
                    ) : (
                      allTasks.filter(t => t.status === 'completed').map(task => {
                        const res = task.results?.find(r => r.teamId === selectedTeamModal.id);
                        const isHunterRestricted = currentUser.role === 'hunter' && currentUser.tribeId !== selectedTeamModal.id;
                        
                        return (
                          <div key={task.id} className="flex justify-between items-center text-xs bg-slate-950 p-2.5 rounded border border-cyan-900/50">
                            <span className="font-montserrat font-semibold text-cyan-200">{task.name}</span>
                            {res ? (
                              <div className="flex items-center gap-3">
                                {!isHunterRestricted && <span className="font-mono font-bold text-cyan-400">#{res.place}º Lugar</span>}
                                <span className="font-mono text-cyan-300 bg-cyan-950 px-2 py-0.5 rounded border border-cyan-500/30">{isHidden || isHunterRestricted ? '??? XP' : `${res.points} XP`}</span>
                              </div>
                            ) : <span className="text-slate-500 font-mono">Não participou</span>}
                          </div>
                        );
                      })
                    )}
                  </div>
                </div>
              )}

              {isEditing && (
                <div className="flex justify-end gap-3 pt-2">
                  <button onClick={() => setIsEditing(false)} className="px-4 py-2 rounded text-xs font-montserrat font-bold text-cyan-500 hover:text-cyan-300 border border-cyan-900 uppercase">Cancelar</button>
                  <button onClick={handleSaveTeam} className="px-5 py-2 rounded text-xs font-montserrat font-bold text-slate-950 bg-cyan-500 hover:bg-cyan-400 shadow-[0_0_12px_rgba(6,182,212,0.6)] uppercase flex items-center gap-1.5"><Save className="w-4 h-4" /> Salvar Dossiê</button>
                </div>
              )}
            </div>
          </div>
        </div>
      )}
    </div>
  );
};

const TimersTab = ({ timers, toggleTimer, resetTimer, toggleAllTimers, resetAllTimers, teams, currentUser }) => {
  const [displayTimes, setDisplayTimes] = useState({ A: 0, B: 0, C: 0, D: 0 });
  const requestRef = useRef();

  const updateDisplay = useCallback(() => {
    const now = Date.now();
    setDisplayTimes(prev => {
      const next = { ...prev };
      let changed = false;
      Object.keys(timers).forEach(id => {
        const t = timers[id];
        const calculated = t.isRunning ? t.time + (now - t.lastStart) : t.time;
        if (next[id] !== calculated) {
          next[id] = calculated;
          changed = true;
        }
      });
      return changed ? next : prev;
    });
    requestRef.current = requestAnimationFrame(updateDisplay);
  }, [timers]);

  useEffect(() => {
    requestRef.current = requestAnimationFrame(updateDisplay);
    return () => cancelAnimationFrame(requestRef.current);
  }, [updateDisplay]);

  const formatTime = (ms) => {
    const totalSeconds = Math.floor(ms / 1000);
    const minutes = Math.floor(totalSeconds / 60);
    const seconds = totalSeconds % 60;
    const centiseconds = Math.floor((ms % 1000) / 10);
    return `${String(minutes).padStart(2, '0')}:${String(seconds).padStart(2, '0')}.${String(centiseconds).padStart(2, '0')}`;
  };

  const anyRunning = Object.values(timers).some(t => t.isRunning);

  return (
    <div className="space-y-8 animate-fade-in relative z-10">
      <div className="text-center mb-6">
        <h2 className="text-xl sm:text-2xl font-montserrat font-bold text-cyan-300 uppercase tracking-[0.25em] mb-2 drop-shadow-[0_0_8px_rgba(6,182,212,0.5)]">[CRONÔMETROS DE TRIBOS]</h2>
        <p className="text-cyan-600 font-mono text-xs tracking-widest">❯❯ CONTROLE TEMPORAL DE COMBATE ❮❮</p>
      </div>

      {currentUser.role === 'admin' && (
        <div className="solo-panel p-5 rounded-xl flex flex-wrap items-center justify-center gap-4">
          <button onClick={toggleAllTimers} className={`flex items-center gap-2 px-6 py-3 rounded-lg font-montserrat font-bold text-xs uppercase tracking-widest border transition-all shadow-[0_0_15px_rgba(6,182,212,0.3)] ${anyRunning ? 'bg-amber-950/80 text-amber-300 border-amber-500' : 'bg-cyan-950/80 text-cyan-300 border-cyan-400'}`}>
            {anyRunning ? <><Square className="w-4 h-4" /> Parar Todos</> : <><Play className="w-4 h-4" /> Iniciar Todos</>}
          </button>
          <button onClick={resetAllTimers} className="flex items-center gap-2 px-6 py-3 rounded-lg font-montserrat font-bold text-xs uppercase tracking-widest bg-slate-900 text-cyan-500 border border-cyan-500/40 hover:bg-slate-800 transition-colors">
            <RotateCcw className="w-4 h-4" /> Resetar Todos
          </button>
        </div>
      )}

      <div className="grid grid-cols-1 md:grid-cols-2 gap-8">
        {teams.map(team => {
          const tState = timers[team.id];
          const timeVal = displayTimes[team.id] || 0;
          return (
            <div key={team.id} className="solo-panel rounded-xl p-6 flex flex-col items-center justify-between relative group" style={{ boxShadow: team.fireGlow, borderColor: team.fireBorder }}>
              <div className="absolute top-2 right-2 w-2 h-2 border-t border-r border-cyan-400"></div>
              <h3 className={`text-lg font-montserrat font-bold uppercase tracking-widest mb-4 ${team.text}`}>{team.name}</h3>
              <div className="bg-slate-950 px-8 py-5 rounded-xl border border-cyan-500/30 mb-6 shadow-[inset_0_0_15px_rgba(6,182,212,0.15)]">
                <span className="font-mono text-3xl sm:text-4xl font-bold text-cyan-100 tracking-wider drop-shadow-[0_0_10px_rgba(6,182,212,0.6)]">{formatTime(timeVal)}</span>
              </div>
              {currentUser.role === 'admin' && (
                <div className="flex items-center gap-3 w-full">
                  <button onClick={() => toggleTimer(team.id)} className={`flex-1 flex items-center justify-center gap-2 py-2.5 rounded-lg font-montserrat font-bold text-xs uppercase tracking-widest border transition-all ${tState.isRunning ? 'bg-amber-950/80 text-amber-300 border-amber-500 shadow-[0_0_10px_rgba(245,158,11,0.4)]' : 'bg-cyan-950/80 text-cyan-300 border-cyan-400 shadow-[0_0_10px_rgba(6,182,212,0.4)]'}`}>
                    {tState.isRunning ? <><Square className="w-4 h-4" /> Pausa</> : <><Play className="w-4 h-4" /> Iniciar</>}
                  </button>
                  <button onClick={() => resetTimer(team.id)} className="px-4 py-2.5 rounded-lg font-montserrat font-bold text-xs bg-slate-900 text-cyan-500 border border-cyan-500/40 hover:bg-slate-800 transition-colors">
                    <RotateCcw className="w-4 h-4" />
                  </button>
                </div>
              )}
            </div>
          );
        })}
      </div>
    </div>
  );
};

const AdminTab = ({ pendingTasks, completedTasks, onCreateTask, onDeleteTask, onCompleteTask, showToast, onLogout, teams, settings, onToggleVisibility, onUpdateMessage, penalties, onAddPenalty, onDeletePenalty }) => {
  const [newTaskName, setNewTaskName] = useState('');
  const [evaluatingTaskId, setEvaluatingTaskId] = useState(null);
  const [sysMsg, setSysMsg] = useState(settings.systemMessage || '');
  const [activeAdminSubTab, setActiveAdminSubTab] = useState('provas'); // 'provas' | 'penalidades'

  const [penaltyTeamId, setPenaltyTeamId] = useState('');
  const [penaltyPoints, setPenaltyPoints] = useState('');
  const [penaltyReason, setPenaltyReason] = useState('');
  
  const initialPlacementForm = useMemo(() => [
    { place: 1, teamId: '', points: '' }, { place: 2, teamId: '', points: '' },
    { place: 3, teamId: '', points: '' }, { place: 4, teamId: '', points: '' },
  ], []);
  
  const [placementForm, setPlacementForm] = useState(initialPlacementForm);
  const [confirmModal, setConfirmModal] = useState({ show: false, message: '', onConfirm: null });

  const handleCreate = (e) => {
    e.preventDefault();
    if (!newTaskName.trim()) return;
    onCreateTask(newTaskName);
    setNewTaskName('');
  };

  const handleCreatePenalty = (e) => {
    e.preventDefault();
    if (!penaltyTeamId || !penaltyPoints || !penaltyReason.trim()) {
      showToast('Preencha todos os campos da penalidade.');
      return;
    }
    const pts = Number(penaltyPoints);
    if (isNaN(pts) || pts <= 0) {
      showToast('Insira um valor de pontos válido.');
      return;
    }
    onAddPenalty({ teamId: penaltyTeamId, points: pts, reason: penaltyReason.trim() });
    setPenaltyPoints('');
    setPenaltyReason('');
    setPenaltyTeamId('');
  };

  const triggerDelete = (taskId) => {
    setConfirmModal({
      show: true,
      message: '[ALERTA DO SISTEMA] Deseja realmente expurgar este registro de masmorra?',
      onConfirm: () => {
        onDeleteTask(taskId);
        setConfirmModal({ show: false, message: '', onConfirm: null });
      }
    });
  };

  const triggerDeletePenalty = (penaltyId) => {
    setConfirmModal({
      show: true,
      message: '[ALERTA DO SISTEMA] Deseja anular esta penalidade aplicada?',
      onConfirm: () => {
        onDeletePenalty(penaltyId);
        setConfirmModal({ show: false, message: '', onConfirm: null });
      }
    });
  };

  const handleSaveResults = (taskId) => {
    const selectedTeams = placementForm.map(p => p.teamId).filter(id => id !== '');
    if (selectedTeams.length !== 4 || new Set(selectedTeams).size !== 4) {
      showToast('Atribua tribos únicas para todas as colocações.');
      return;
    }
    if (placementForm.some(p => p.points === '' || isNaN(Number(p.points)))) {
       showToast('Insira pontuações válidas em todas as posições.');
       return;
    }
    const formattedResults = placementForm.map(p => ({ place: p.place, teamId: p.teamId, points: Number(p.points) }));
    onCompleteTask(taskId, formattedResults);
    setEvaluatingTaskId(null);
    setPlacementForm(initialPlacementForm);
  };

  return (
    <div className="max-w-4xl mx-auto space-y-8 animate-fade-in relative z-10">
      <div className="flex justify-between items-center solo-panel p-5 rounded-xl">
        <div className="flex items-center gap-4">
          <div className="bg-cyan-950/60 p-2.5 rounded-lg border border-cyan-500/40 shadow-[0_0_10px_rgba(6,182,212,0.3)]"><Lock className="w-4 h-4 text-cyan-400" /></div>
          <div>
            <h2 className="font-montserrat font-bold text-cyan-300 uppercase tracking-widest text-sm">[SISTEMA ADMINISTRATIVO ATIVO]</h2>
            <p className="text-xs text-cyan-600 font-mono mt-1">Nível de Acesso: Monarca</p>
          </div>
        </div>
        <button onClick={onLogout} className="flex items-center gap-2 text-red-400 hover:text-red-200 hover:bg-red-950/50 px-4 py-2 rounded-lg font-montserrat font-bold text-xs uppercase tracking-widest transition-colors border border-red-500/30"><LogOut className="w-4 h-4" /> Sair de Sessão</button>
      </div>

      <div className="flex flex-wrap gap-3 border-b border-cyan-500/20 pb-4">
        <button onClick={() => setActiveAdminSubTab('provas')} className={`px-5 py-2.5 rounded-lg font-montserrat font-bold text-xs uppercase tracking-widest border transition-all ${activeAdminSubTab === 'provas' ? 'bg-cyan-950/80 text-cyan-300 border-cyan-400 shadow-[0_0_12px_rgba(6,182,212,0.4)]' : 'bg-slate-900/40 text-cyan-600 border-cyan-500/20 hover:text-cyan-400'}`}>
          Gerenciar Provas
        </button>
        <button onClick={() => setActiveAdminSubTab('concluidas')} className={`px-5 py-2.5 rounded-lg font-montserrat font-bold text-xs uppercase tracking-widest border transition-all ${activeAdminSubTab === 'concluidas' ? 'bg-cyan-950/80 text-cyan-300 border-cyan-400 shadow-[0_0_12px_rgba(6,182,212,0.4)]' : 'bg-slate-900/40 text-cyan-600 border-cyan-500/20 hover:text-cyan-400'}`}>
          Provas Concluídas
        </button>
        <button onClick={() => setActiveAdminSubTab('penalidades')} className={`px-5 py-2.5 rounded-lg font-montserrat font-bold text-xs uppercase tracking-widest border transition-all ${activeAdminSubTab === 'penalidades' ? 'bg-red-950/80 text-red-300 border-red-500 shadow-[0_0_12px_rgba(239,68,68,0.4)]' : 'bg-slate-900/40 text-cyan-600 border-cyan-500/20 hover:text-cyan-400'}`}>
          Gerenciar Penalidades
        </button>
      </div>

      {activeAdminSubTab === 'provas' && (
        <>
          <div className="solo-panel p-5 sm:p-6 rounded-xl flex flex-col sm:flex-row items-start sm:items-center justify-between gap-4 border-l-2 border-l-cyan-400" style={{ boxShadow: '0 0 20px rgba(6, 182, 212, 0.15)' }}>
            <div>
              <h3 className="text-sm font-montserrat font-bold text-cyan-300 mb-1 flex items-center gap-2 uppercase tracking-widest"><Eye className="w-4 h-4 text-cyan-400" /> Visibilidade de Classificação</h3>
              <p className="text-xs text-cyan-600 font-mono">Controla se Líderes e Caçadores podem ver os pontos e o ranking.</p>
            </div>
            <button 
              onClick={onToggleVisibility} 
              className={`flex items-center gap-2 px-5 py-2.5 rounded-lg font-montserrat font-bold text-xs transition-all uppercase tracking-widest border shadow-[0_0_10px_rgba(6,182,212,0.2)] ${settings.isRankingVisible ? 'bg-cyan-600/80 text-slate-950 border-cyan-400 hover:bg-cyan-500' : 'bg-slate-900 text-cyan-500 border-cyan-500/40 hover:bg-slate-800'}`}
            >
              {settings.isRankingVisible ? <><Eye className="w-4 h-4" /> PÚBLICO: REVELADO</> : <><EyeOff className="w-4 h-4" /> PÚBLICO: OCULTO</>}
            </button>
          </div>

          <div className="solo-panel p-5 sm:p-6 rounded-xl relative border-l-2 border-l-cyan-400 mt-8">
            <h3 className="text-sm font-montserrat font-bold text-cyan-300 mb-4 flex items-center gap-2 uppercase tracking-widest"><MessageSquare className="w-4 h-4 text-cyan-400" /> Mensagem do Sistema</h3>
            <textarea value={sysMsg} onChange={(e) => setSysMsg(e.target.value)} className="w-full px-4 py-3 rounded-lg bg-slate-950 border border-cyan-500/40 text-cyan-200 focus:ring-1 focus:ring-cyan-400 outline-none font-inter resize-none h-24 mb-4 shadow-[inset_0_0_8px_rgba(6,182,212,0.1)]" placeholder="Deixe um recado, regra ou dica para os Caçadores..." />
            <div className="flex justify-end">
                <button onClick={() => onUpdateMessage(sysMsg)} className="bg-cyan-600/80 hover:bg-cyan-500 border border-cyan-400 text-slate-950 font-montserrat font-bold text-xs py-2.5 px-5 rounded-lg transition-colors flex items-center gap-2 uppercase tracking-widest shadow-[0_0_12px_rgba(6,182,212,0.3)]"><Save className="w-4 h-4" /> Atualizar Mural</button>
            </div>
          </div>

          <div className="solo-panel p-6 sm:p-8 rounded-xl relative mt-8">
            <div className="absolute top-0 left-0 w-2 h-2 border-t-2 border-l-2 border-cyan-400"></div>
            <div className="absolute top-0 right-0 w-2 h-2 border-t-2 border-r-2 border-cyan-400"></div>
            <h3 className="text-sm font-montserrat font-bold text-cyan-300 mb-5 flex items-center gap-2 uppercase tracking-widest"><Plus className="w-4 h-4 text-cyan-400" /> Criar Nova Prova</h3>
            <form onSubmit={handleCreate} className="flex flex-col sm:flex-row gap-4">
              <input type="text" value={newTaskName} onChange={(e) => setNewTaskName(e.target.value)} placeholder="Nome da Prova (Ex: Prova 1)" className="flex-1 px-4 py-3 rounded-lg bg-slate-950 border border-cyan-500/40 text-cyan-200 placeholder-cyan-700 focus:ring-1 focus:ring-cyan-400 outline-none font-inter shadow-[inset_0_0_8px_rgba(6,182,212,0.2)]" required />
              <button type="submit" className="bg-cyan-600/80 hover:bg-cyan-500 border border-cyan-400 text-slate-950 font-montserrat font-bold text-xs py-3 px-6 rounded-lg transition-colors whitespace-nowrap uppercase tracking-widest shadow-[0_0_12px_rgba(6,182,212,0.4)]">Cadastrar</button>
            </form>
          </div>

          <div>
            <h3 className="text-sm font-montserrat font-bold text-cyan-300 mb-5 uppercase tracking-widest">[PROVAS PENDENTES]</h3>
            {pendingTasks.length === 0 ? (
              <p className="text-cyan-600 font-mono text-xs solo-panel p-6 rounded-xl text-center uppercase tracking-widest">Nenhuma prova pendente.</p>
            ) : (
              <div className="space-y-4">
                {pendingTasks.map(task => (
                  <div key={task.id} className="solo-panel rounded-xl overflow-hidden border-l-2 border-l-cyan-400" style={{ boxShadow: '0 0 20px rgba(6, 182, 212, 0.15)' }}>
                    <div className="p-5 sm:p-6">
                      <div className="flex justify-between items-start mb-5">
                        <h3 className="text-base sm:text-lg font-montserrat font-bold text-cyan-100 uppercase tracking-wide">{task.name}</h3>
                        <button onClick={() => triggerDelete(task.id)} className="text-cyan-500 hover:text-cyan-300 p-2 hover:bg-cyan-950/40 rounded-lg transition-colors border border-cyan-900/40"><Trash2 className="w-4 h-4" /></button>
                      </div>
                      {evaluatingTaskId === task.id ? (
                        <div className="bg-slate-900/90 p-5 sm:p-7 rounded-lg border border-cyan-500/40 animate-fade-in shadow-[0_0_15px_rgba(6,182,212,0.2)]">
                          <div className="flex justify-between items-center mb-6">
                            <h4 className="font-montserrat font-bold text-sm text-cyan-300 flex items-center gap-2 uppercase tracking-widest"><Award className="w-4 h-4" /> Atribuir Colocação</h4>
                            <button onClick={() => { setEvaluatingTaskId(null); setPlacementForm(initialPlacementForm); }} className="text-cyan-500 hover:text-cyan-300"><X className="w-5 h-5" /></button>
                          </div>
                          <div className="space-y-3">
                            {placementForm.map((row, index) => (
                              <div key={row.place} className="flex flex-col sm:flex-row items-center gap-3 bg-slate-950 p-3 rounded-lg border border-cyan-900/60">
                                <div className="font-montserrat font-bold text-cyan-400 w-full sm:w-16 text-center text-sm">#{row.place}</div>
                                <select value={row.teamId} onChange={(e) => { const n = [...placementForm]; n[index].teamId = e.target.value; setPlacementForm(n); }} className="flex-1 px-3 py-2.5 rounded-lg bg-slate-900 border border-cyan-500/40 text-cyan-200 outline-none font-inter text-sm w-full">
                                  <option value="">Selecionar Tribo...</option>
                                  {teams.map(team => <option key={team.id} value={team.id} disabled={placementForm.some((p, i) => i !== index && p.teamId === team.id)}>{team.name}</option>)}
                                </select>
                                <input type="number" value={row.points} onChange={(e) => { const n = [...placementForm]; n[index].points = e.target.value; setPlacementForm(n); }} placeholder="XP" className="w-full sm:w-32 px-3 py-2.5 rounded-lg bg-slate-900 border border-cyan-500/40 text-cyan-200 outline-none font-mono text-center text-sm" />
                              </div>
                            ))}
                          </div>
                          <div className="mt-6 flex justify-end">
                            <button onClick={() => handleSaveResults(task.id)} className="bg-cyan-600/80 hover:bg-cyan-500 border border-cyan-400 text-slate-950 font-montserrat font-bold text-xs py-3 px-6 rounded-lg transition-colors flex items-center gap-2 justify-center uppercase tracking-widest shadow-[0_0_12px_rgba(6,182,212,0.4)]"><Save className="w-4 h-4" /> Concluir</button>
                          </div>
                        </div>
                      ) : (
                        <button onClick={() => { setEvaluatingTaskId(task.id); setPlacementForm(initialPlacementForm); }} className="w-full bg-slate-900/60 hover:bg-slate-800 text-cyan-300 border border-cyan-500/40 font-montserrat font-bold py-3.5 px-4 rounded-lg flex justify-center items-center gap-2 uppercase tracking-widest shadow-[0_0_10px_rgba(6,182,212,0.2)]"><Award className="w-4 h-4" /> Registrar Resultados</button>
                      )}
                    </div>
                  </div>
                ))}
              </div>
            )}
          </div>
        </>
      )}
      
      {activeAdminSubTab === 'concluidas' && (
        <div className="space-y-8 animate-fade-in">
          <h3 className="text-sm font-montserrat font-bold text-cyan-300 mb-5 uppercase tracking-widest">[PROVAS CONCLUÍDAS]</h3>
          {completedTasks.length === 0 ? (
            <p className="text-cyan-600 font-mono text-xs solo-panel p-6 rounded-xl text-center uppercase tracking-widest">Nenhuma prova concluída ainda.</p>
          ) : (
            <div className="space-y-4">
              {completedTasks.map(task => (
                <div key={task.id} className="solo-panel rounded-xl overflow-hidden border-l-2 border-l-cyan-400" style={{ boxShadow: '0 0 20px rgba(6, 182, 212, 0.15)' }}>
                  <div className="p-5 sm:p-6">
                    <div className="flex justify-between items-center mb-4">
                      <h3 className="text-base sm:text-lg font-montserrat font-bold text-cyan-100 uppercase tracking-wide">{task.name}</h3>
                      <button onClick={() => triggerDelete(task.id)} className="text-red-400 hover:text-red-300 p-2 hover:bg-red-950/40 rounded-lg transition-colors border border-red-900/40" title="Excluir Prova"><Trash2 className="w-4 h-4" /></button>
                    </div>
                    <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-3 bg-slate-950/50 p-4 rounded-lg border border-cyan-900/30">
                      {task.results && [...task.results].sort((a,b) => a.place - b.place).map(result => {
                        const team = teams.find(t => t.id === result.teamId);
                        if (!team) return null;
                        return (
                          <div key={result.place} className={`flex items-center gap-3 p-2 rounded-lg border-l-2 ${team.lightBg} border-l-cyan-400 border border-cyan-900/40`}>
                            <div className="font-montserrat font-bold text-lg text-cyan-600">#{result.place}</div>
                            <div>
                              <div className={`font-inter font-bold text-[10px] uppercase tracking-wider ${team.text}`}>{team.name}</div>
                              <div className="text-cyan-200 font-mono text-xs">{result.points} XP</div>
                            </div>
                          </div>
                        );
                      })}
                    </div>
                  </div>
                </div>
              ))}
            </div>
          )}
        </div>
      )}

      {activeAdminSubTab === 'penalidades' && (
        <div className="space-y-8 animate-fade-in">
          <div className="solo-panel p-6 sm:p-8 rounded-xl relative border-l-2 border-l-red-500" style={{ boxShadow: '0 0 20px rgba(239, 68, 68, 0.15)' }}>
            <h3 className="text-sm font-montserrat font-bold text-red-300 mb-5 flex items-center gap-2 uppercase tracking-widest"><AlertTriangle className="w-4 h-4 text-red-400" /> Aplicar Penalidade (-XP)</h3>
            <form onSubmit={handleCreatePenalty} className="space-y-4">
              <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
                <div>
                  <label className="block text-xs font-montserrat font-bold text-cyan-400 uppercase tracking-widest mb-2">Tribo Alvo</label>
                  <select value={penaltyTeamId} onChange={(e) => setPenaltyTeamId(e.target.value)} className="w-full px-4 py-3 rounded-lg bg-slate-950 border border-red-500/40 text-cyan-200 outline-none font-inter text-sm" required>
                    <option value="">Selecionar Tribo...</option>
                    {teams.map(team => <option key={team.id} value={team.id}>{team.name}</option>)}
                  </select>
                </div>
                <div>
                  <label className="block text-xs font-montserrat font-bold text-cyan-400 uppercase tracking-widest mb-2">Pontos a Retirar (XP)</label>
                  <input type="number" min="1" value={penaltyPoints} onChange={(e) => setPenaltyPoints(e.target.value)} placeholder="Ex: 50" className="w-full px-4 py-3 rounded-lg bg-slate-950 border border-red-500/40 text-red-300 outline-none font-mono text-sm" required />
                </div>
              </div>
              <div>
                <label className="block text-xs font-montserrat font-bold text-cyan-400 uppercase tracking-widest mb-2">Motivo da Penalidade</label>
                <input type="text" value={penaltyReason} onChange={(e) => setPenaltyReason(e.target.value)} placeholder="Ex: Desrespeito às regras da masmorra" className="w-full px-4 py-3 rounded-lg bg-slate-950 border border-red-500/40 text-cyan-200 outline-none font-inter text-sm" required />
              </div>
              <div className="flex justify-end">
                <button type="submit" className="bg-red-600/80 hover:bg-red-500 border border-red-400 text-slate-950 font-montserrat font-bold text-xs py-3 px-6 rounded-lg transition-colors uppercase tracking-widest shadow-[0_0_12px_rgba(239,68,68,0.4)]">Aplicar Punição</button>
              </div>
            </form>
          </div>

          <div>
            <h3 className="text-sm font-montserrat font-bold text-red-300 mb-5 uppercase tracking-widest">[HISTÓRICO DE PUNIÇÕES]</h3>
            {penalties.length === 0 ? (
              <p className="text-cyan-600 font-mono text-xs solo-panel p-6 rounded-xl text-center uppercase tracking-widest">Nenhuma penalidade aplicada até o momento.</p>
            ) : (
              <div className="space-y-3">
                {penalties.map(p => {
                  const team = teams.find(t => t.id === p.teamId);
                  return (
                    <div key={p.id} className="solo-panel p-4 rounded-xl flex items-center justify-between border border-red-500/30">
                      <div>
                        <div className="flex items-center gap-3 mb-1">
                          <span className={`font-montserrat font-bold text-sm uppercase tracking-wider ${team ? team.text : 'text-red-400'}`}>{team ? team.name : 'Tribo Desconhecida'}</span>
                          <span className="bg-red-950/80 text-red-400 font-mono font-bold text-xs px-2 py-0.5 rounded border border-red-500/40">-{p.points} XP</span>
                        </div>
                        <p className="text-cyan-200 font-inter text-xs">Motivo: {p.reason}</p>
                      </div>
                      <button onClick={() => triggerDeletePenalty(p.id)} className="text-red-400 hover:text-red-200 p-2 hover:bg-red-950/50 rounded-lg transition-colors border border-red-900/40"><Trash2 className="w-4 h-4" /></button>
                    </div>
                  );
                })}
              </div>
            )}
          </div>
        </div>
      )}

      {confirmModal.show && (
        <div className="fixed inset-0 bg-slate-950/85 backdrop-blur-sm z-50 flex items-center justify-center p-4 animate-fade-in">
          <div className="solo-panel rounded-xl p-6 max-w-sm w-full shadow-2xl animate-fade-in-up border border-cyan-500/60 relative">
            <h3 className="text-sm font-montserrat font-bold text-cyan-300 mb-3 uppercase tracking-widest flex items-center gap-2"><AlertTriangle className="w-4 h-4 text-cyan-400" /> [ALERTA DO SISTEMA]</h3>
            <p className="text-cyan-600 font-inter text-sm mb-8">{confirmModal.message}</p>
            <div className="flex gap-3 justify-end">
              <button onClick={() => setConfirmModal({ show: false, message: '', onConfirm: null })} className="px-4 py-2 rounded font-montserrat font-bold text-xs text-cyan-500 hover:text-cyan-300 hover:bg-cyan-950/40 border border-cyan-900 uppercase">Cancelar</button>
              <button onClick={confirmModal.onConfirm} className="px-5 py-2 rounded font-montserrat font-bold text-xs text-slate-950 bg-cyan-500 hover:bg-cyan-400 shadow-[0_0_12px_rgba(6,182,212,0.6)] uppercase">Confirmar</button>
            </div>
          </div>
        </div>
      )}
    </div>
  );
};

// ==========================================
// COMPONENTE PRINCIPAL (MAIN APP)
// ==========================================
export default function App() {
  const [activeTab, setActiveTab] = useState('ranking');
  const [currentUser, setCurrentUser] = useState(null); 
  const [systemPassword, setSystemPassword] = useState('');
  
  const [user, setUser] = useState(null); 
  const [tasks, setTasks] = useState([]);
  const [penalties, setPenalties] = useState([]);
  const [teams, setTeams] = useState(DEFAULT_TEAM_CONFIG);
  const [settings, setSettings] = useState({ isRankingVisible: true, systemMessage: 'O Sistema aguarda o despertar de vocês.' });
  
  const [toast, setToast] = useState({ show: false, message: '', type: 'error' });
  const [timers, setTimers] = useState({
    A: { time: 0, isRunning: false, lastStart: 0 }, B: { time: 0, isRunning: false, lastStart: 0 },
    C: { time: 0, isRunning: false, lastStart: 0 }, D: { time: 0, isRunning: false, lastStart: 0 },
  });

  const showToast = useCallback((message, type = 'error') => {
    setToast({ show: true, message, type });
    setTimeout(() => setToast({ show: false, message: '', type: 'error' }), 3500);
  }, []);

  useEffect(() => {
    if (!hasFirebase || !auth) return;
    const initAuth = async () => {
      if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
        await signInWithCustomToken(auth, __initial_auth_token);
      } else { await signInAnonymously(auth); }
    };
    initAuth();
    const unsubscribe = onAuthStateChanged(auth, setUser);
    return () => unsubscribe();
  }, []);

  useEffect(() => {
    if (hasFirebase && user && db) {
      const unsubscribeTasks = onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'tasks'), (snapshot) => {
        setTasks(snapshot.docs.map(doc => doc.data()).sort((a, b) => Number(a.id) - Number(b.id)));
      });

      const unsubscribePenalties = onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'penalties'), (snapshot) => {
        setPenalties(snapshot.docs.map(doc => doc.data()).sort((a, b) => Number(b.id) - Number(a.id)));
      });
      
      const unsubscribeTeams = onSnapshot(doc(db, 'artifacts', appId, 'public', 'data', 'config', 'teams'), (docSnap) => {
        if (docSnap.exists()) {
          const dbTeams = docSnap.data().teams;
          const mergedTeams = dbTeams.map(pt => {
             const def = DEFAULT_TEAM_CONFIG.find(d => d.id === pt.id) || {};
             const rawParticipants = pt.participants || def.participants || [];
             const normalizedParticipants = rawParticipants.map(p => typeof p === 'string' ? { name: p, role: 'Caçador' } : p);
             return { ...pt, color: def.color || pt.color, text: def.text || pt.text, lightBg: def.lightBg || pt.lightBg, fireGlow: def.fireGlow || pt.fireGlow, fireBorder: def.fireBorder || pt.fireBorder, participants: normalizedParticipants };
          });
          setTeams(mergedTeams);
        }
      });

      const unsubscribeSettings = onSnapshot(doc(db, 'artifacts', appId, 'public', 'data', 'config', 'settings'), (docSnap) => {
        if (docSnap.exists()) setSettings(docSnap.data());
      });

      // Firebase Sync for Timers
      const unsubscribeTimers = onSnapshot(doc(db, 'artifacts', appId, 'public', 'data', 'config', 'timers'), (docSnap) => {
        if (docSnap.exists()) setTimers(docSnap.data().timers);
      });
      
      return () => { unsubscribeTasks(); unsubscribePenalties(); unsubscribeTeams(); unsubscribeSettings(); unsubscribeTimers(); };
    } else if (!hasFirebase) {
      const savedTasks = localStorage.getItem('gincana_tasks_local');
      if (savedTasks) setTasks(JSON.parse(savedTasks));

      const savedPenalties = localStorage.getItem('gincana_penalties_local');
      if (savedPenalties) setPenalties(JSON.parse(savedPenalties));
      
      const savedTeams = localStorage.getItem('gincana_teams_local');
      if (savedTeams) {
         const parsed = JSON.parse(savedTeams);
         const mergedTeams = parsed.map(pt => {
             const def = DEFAULT_TEAM_CONFIG.find(d => d.id === pt.id) || {};
             const rawParticipants = pt.participants || def.participants || [];
             const normalizedParticipants = rawParticipants.map(p => typeof p === 'string' ? { name: p, role: 'Caçador' } : p);
             return { ...pt, color: def.color || pt.color, text: def.text || pt.text, lightBg: def.lightBg || pt.lightBg, fireGlow: def.fireGlow || pt.fireGlow, fireBorder: def.fireBorder || pt.fireBorder, participants: normalizedParticipants };
         });
         setTeams(mergedTeams);
      }

      const savedSettings = localStorage.getItem('gincana_settings_local');
      if (savedSettings) setSettings(JSON.parse(savedSettings));

      const savedTimers = localStorage.getItem('gincana_timers_local');
      if (savedTimers) setTimers(JSON.parse(savedTimers));
    }
  }, [user]);

  useEffect(() => {
    if (!hasFirebase) {
      localStorage.setItem('gincana_tasks_local', JSON.stringify(tasks));
      localStorage.setItem('gincana_penalties_local', JSON.stringify(penalties));
      localStorage.setItem('gincana_teams_local', JSON.stringify(teams));
      localStorage.setItem('gincana_settings_local', JSON.stringify(settings));
      localStorage.setItem('gincana_timers_local', JSON.stringify(timers));
    }
  }, [tasks, penalties, teams, settings, timers]);

  const teamScores = useMemo(() => {
    return teams.map(team => {
      let points = 0, tasksCompleted = 0;
      tasks.filter(t => t.status === 'completed').forEach(task => {
        const res = task.results?.find(r => r.teamId === team.id);
        if (res) { points += res.points; tasksCompleted += 1; }
      });

      penalties.filter(p => p.teamId === team.id).forEach(p => {
        points -= p.points;
      });

      return { ...team, points, average: tasksCompleted > 0 ? (points / tasksCompleted).toFixed(1) : '0.0', tasksCompleted };
    }).sort((a, b) => b.points - a.points);
  }, [tasks, penalties, teams]);

  const completedTasks = useMemo(() => tasks.filter(t => t.status === 'completed').reverse(), [tasks]);
  const pendingTasks = useMemo(() => tasks.filter(t => t.status === 'pending'), [tasks]);

  const handleLogin = (e) => {
    e.preventDefault();
    const pwd = systemPassword.trim().toLowerCase();
    if (pwd === 'admin123') {
      setCurrentUser({ role: 'admin', tribeId: null });
      setActiveTab('ranking');
      showToast('SISTEMA DESBLOQUEADO: NÍVEL MONARCA', 'success');
    } else if (pwd === 'lidergarden') {
      setCurrentUser({ role: 'leader', tribeId: 'A' });
      setActiveTab('ranking');
      showToast('ACESSO CONCEDIDO: LÍDER GARDEN', 'success');
    } else if (pwd === 'liderflock') {
      setCurrentUser({ role: 'leader', tribeId: 'B' });
      setActiveTab('ranking');
      showToast('ACESSO CONCEDIDO: LÍDER FLOCK', 'success');
    } else if (pwd === 'liderbold') {
      setCurrentUser({ role: 'leader', tribeId: 'C' });
      setActiveTab('ranking');
      showToast('ACESSO CONCEDIDO: LÍDER BOLD', 'success');
    } else if (pwd === 'liderburn') {
      setCurrentUser({ role: 'leader', tribeId: 'D' });
      setActiveTab('ranking');
      showToast('ACESSO CONCEDIDO: LÍDER BURN', 'success');
    } else if (pwd === 'cacador' || pwd === 'convidado') {
      setCurrentUser({ role: 'hunter', tribeId: null });
      setActiveTab('ranking');
      showToast('ACESSO CONCEDIDO: CAÇADOR', 'success');
    } else {
      showToast('SENHA DE ACESSO INVÁLIDA');
    }
    setSystemPassword('');
  };

  const handleLogout = () => {
    setCurrentUser(null);
    setActiveTab('ranking');
    showToast('SESSÃO ENCERRADA PELO SISTEMA', 'success');
  };

  const handleCreateTask = useCallback(async (name) => {
    const newTask = { id: Date.now().toString(), name, status: 'pending', results: [] };
    if (hasFirebase && user && db) {
      await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'tasks', newTask.id), newTask);
    } else {
      setTasks(prev => [...prev, newTask]);
    }
    showToast('MASMORRA CADASTRADA COM SUCESSO', 'success');
  }, [user, showToast]);

  const handleDeleteTask = useCallback(async (id) => {
    if (hasFirebase && user && db) {
      await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'tasks', id));
    } else {
      setTasks(prev => prev.filter(t => t.id !== id));
    }
    showToast('MASMORRA EXPURGADA', 'success');
  }, [user, showToast]);

  const handleCompleteTask = useCallback(async (id, results) => {
    if (hasFirebase && user && db) {
      const taskRef = doc(db, 'artifacts', appId, 'public', 'data', 'tasks', id);
      await setDoc(taskRef, { status: 'completed', results }, { merge: true });
    } else {
      setTasks(prev => prev.map(t => t.id === id ? { ...t, status: 'completed', results } : t));
    }
    showToast('COLOCAÇÃO REGISTRADA COM SUCESSO', 'success');
  }, [user, showToast]);

  const handleUpdateTeam = useCallback(async (updatedTeam) => {
    const newTeams = teams.map(t => t.id === updatedTeam.id ? updatedTeam : t);
    setTeams(newTeams);
    if (hasFirebase && user && db) {
      await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'config', 'teams'), { teams: newTeams });
    }
    showToast('DOSSIÊ DA GUILDA ATUALIZADO', 'success');
  }, [teams, user, showToast]);

  const handleToggleVisibility = useCallback(async () => {
    const nextVal = !settings.isRankingVisible;
    const newSettings = { ...settings, isRankingVisible: nextVal };
    setSettings(newSettings);
    if (hasFirebase && user && db) {
      await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'config', 'settings'), newSettings);
    }
    showToast(nextVal ? 'CLASSIFICAÇÃO PÚBLICA REVELADA' : 'CLASSIFICAÇÃO PÚBLICA OCULTADA', 'success');
  }, [settings, user, showToast]);

  const handleUpdateMessage = useCallback(async (msg) => {
    const newSettings = { ...settings, systemMessage: msg };
    setSettings(newSettings);
    if (hasFirebase && user && db) {
      await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'config', 'settings'), newSettings);
    }
    showToast('MURAL DO SISTEMA ATUALIZADO', 'success');
  }, [settings, user, showToast]);

  const handleAddPenalty = useCallback(async (penaltyData) => {
    const newPenalty = { id: Date.now().toString(), ...penaltyData };
    if (hasFirebase && user && db) {
      await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'penalties', newPenalty.id), newPenalty);
    } else {
      setPenalties(prev => [newPenalty, ...prev]);
    }
    showToast('PENALIDADE APLICADA COM SUCESSO', 'success');
  }, [user, showToast]);

  const handleDeletePenalty = useCallback(async (id) => {
    if (hasFirebase && user && db) {
      await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'penalties', id));
    } else {
      setPenalties(prev => prev.filter(p => p.id !== id));
    }
    showToast('PENALIDADE ANULADA', 'success');
  }, [user, showToast]);

  // Firebase synced Timer Controls
  const toggleTimer = async (id) => {
    const newTimers = { ...timers };
    const t = newTimers[id];
    const now = Date.now();
    if (!t.isRunning) {
      newTimers[id] = { ...t, isRunning: true, lastStart: now };
    } else {
      const elapsed = t.time + (now - t.lastStart);
      newTimers[id] = { ...t, isRunning: false, time: elapsed };
    }
    
    setTimers(newTimers);
    if (hasFirebase && user && db) {
      await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'config', 'timers'), { timers: newTimers });
    }
  };

  const resetTimer = async (id) => {
    const newTimers = { ...timers, [id]: { time: 0, isRunning: false, lastStart: 0 } };
    setTimers(newTimers);
    if (hasFirebase && user && db) {
      await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'config', 'timers'), { timers: newTimers });
    }
  };

  const toggleAllTimers = async () => {
    const anyRunning = Object.values(timers).some(t => t.isRunning);
    const now = Date.now();
    const newTimers = {};
    
    Object.keys(timers).forEach(id => {
      const t = timers[id];
      if (!anyRunning) {
        newTimers[id] = { ...t, isRunning: true, lastStart: now };
      } else {
        const elapsed = t.isRunning ? t.time + (now - t.lastStart) : t.time;
        newTimers[id] = { ...t, isRunning: false, time: elapsed };
      }
    });
    
    setTimers(newTimers);
    if (hasFirebase && user && db) {
      await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'config', 'timers'), { timers: newTimers });
    }
  };

  const resetAllTimers = async () => {
    const newTimers = {
      A: { time: 0, isRunning: false, lastStart: 0 }, B: { time: 0, isRunning: false, lastStart: 0 },
      C: { time: 0, isRunning: false, lastStart: 0 }, D: { time: 0, isRunning: false, lastStart: 0 },
    };
    setTimers(newTimers);
    if (hasFirebase && user && db) {
      await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'config', 'timers'), { timers: newTimers });
    }
  };

  return (
    <div className="min-h-screen bg-slate-950 text-cyan-100 font-inter relative overflow-hidden select-none">
      <div className="absolute inset-0 bg-[radial-gradient(circle_at_50%_0%,rgba(6,182,212,0.15),transparent_70%)] pointer-events-none"></div>
      <div className="absolute inset-0 lightning-overlay pointer-events-none"></div>

      {!currentUser ? (
        <div className="min-h-screen flex items-center justify-center p-4 relative z-10">
          <div className="solo-panel p-8 sm:p-12 rounded-2xl max-w-md w-full relative animate-fade-in-up">
            <div className="absolute top-0 left-0 w-3 h-3 border-t-2 border-l-2 border-cyan-400"></div>
            <div className="absolute top-0 right-0 w-3 h-3 border-t-2 border-r-2 border-cyan-400"></div>
            <div className="absolute bottom-0 left-0 w-3 h-3 border-b-2 border-l-2 border-cyan-400"></div>
            <div className="absolute bottom-0 right-0 w-3 h-3 border-b-2 border-r-2 border-cyan-400"></div>

            <div className="text-center mb-8">
              <Zap className="w-12 h-12 text-cyan-400 mx-auto mb-3 drop-shadow-[0_0_12px_rgba(6,182,212,0.8)] animate-pulse" />
              <h1 className="text-xl sm:text-2xl font-montserrat font-bold text-cyan-300 tracking-[0.25em] uppercase drop-shadow-[0_0_8px_rgba(6,182,212,0.5)]">TRIBOFULL SYSTEM</h1>
              <p className="text-cyan-600/80 font-mono text-xs uppercase tracking-widest mt-2">❯❯ PROTOCOLO DE AUTENTICAÇÃO ❮❮</p>
            </div>

            <form onSubmit={handleLogin} className="space-y-6">
              <div>
                <label className="block text-xs font-montserrat font-bold text-cyan-400 uppercase tracking-widest mb-2">Chave de Acesso</label>
                <input type="password" value={systemPassword} onChange={(e) => setSystemPassword(e.target.value)} placeholder="Digite sua senha de acesso..." className="w-full px-4 py-3 rounded-lg bg-slate-950 border border-cyan-500/40 text-cyan-200 outline-none font-mono text-sm shadow-[inset_0_0_8px_rgba(6,182,212,0.2)] focus:ring-1 focus:ring-cyan-400" required />
              </div>
              <button type="submit" className="w-full bg-cyan-600/80 hover:bg-cyan-500 border border-cyan-400 text-slate-950 font-montserrat font-bold text-xs py-3.5 rounded-lg transition-colors uppercase tracking-[0.2em] shadow-[0_0_15px_rgba(6,182,212,0.4)]">
                Despertar Sistema
              </button>
            </form>
          </div>
        </div>
      ) : (
        <div className="min-h-screen flex flex-col">
          <header className="solo-panel border-b border-cyan-500/30 sticky top-0 z-40">
            <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 h-20 flex items-center justify-between">
              <div className="flex items-center gap-3">
                <Zap className="w-6 h-6 text-cyan-400 drop-shadow-[0_0_8px_rgba(6,182,212,0.8)]" />
                <span className="font-montserrat font-bold text-base sm:text-lg tracking-[0.2em] text-cyan-200 uppercase drop-shadow-[0_0_6px_rgba(6,182,212,0.4)]">TRIBOFULL</span>
              </div>

              <nav className="hidden md:flex items-center gap-2">
                <button onClick={() => setActiveTab('ranking')} className={`px-4 py-2 rounded-lg font-montserrat font-bold text-xs uppercase tracking-widest transition-all ${activeTab === 'ranking' ? 'bg-cyan-950/80 text-cyan-300 border border-cyan-400 shadow-[0_0_12px_rgba(6,182,212,0.4)]' : 'text-cyan-600 hover:text-cyan-300'}`}>
                  Ranking de Tribos
                </button>
                <button onClick={() => setActiveTab('quadro')} className={`px-4 py-2 rounded-lg font-montserrat font-bold text-xs uppercase tracking-widest transition-all ${activeTab === 'quadro' ? 'bg-cyan-950/80 text-cyan-300 border border-cyan-400 shadow-[0_0_12px_rgba(6,182,212,0.4)]' : 'text-cyan-600 hover:text-cyan-300'}`}>
                  Quadro
                </button>
                <button onClick={() => setActiveTab('tribos')} className={`px-4 py-2 rounded-lg font-montserrat font-bold text-xs uppercase tracking-widest transition-all ${activeTab === 'tribos' ? 'bg-cyan-950/80 text-cyan-300 border border-cyan-400 shadow-[0_0_12px_rgba(6,182,212,0.4)]' : 'text-cyan-600 hover:text-cyan-300'}`}>
                  Status de Tribos
                </button>
                {currentUser.role === 'admin' && (
                  <button onClick={() => setActiveTab('timers')} className={`px-4 py-2 rounded-lg font-montserrat font-bold text-xs uppercase tracking-widest transition-all ${activeTab === 'timers' ? 'bg-cyan-950/80 text-cyan-300 border border-cyan-400 shadow-[0_0_12px_rgba(6,182,212,0.4)]' : 'text-cyan-600 hover:text-cyan-300'}`}>
                    Relógios
                  </button>
                )}
                {currentUser.role === 'admin' && (
                  <button onClick={() => setActiveTab('admin')} className={`px-4 py-2 rounded-lg font-montserrat font-bold text-xs uppercase tracking-widest transition-all ${activeTab === 'admin' ? 'bg-cyan-950/80 text-cyan-300 border border-cyan-400 shadow-[0_0_12px_rgba(6,182,212,0.4)]' : 'text-cyan-600 hover:text-cyan-300'}`}>
                    Provas
                  </button>
                )}
              </nav>

              <button onClick={handleLogout} className="flex items-center gap-2 text-red-400 hover:text-red-200 px-3 py-2 rounded font-montserrat font-bold text-xs uppercase tracking-widest border border-red-500/30">
                <LogOut className="w-4 h-4" /> Sair
              </button>
            </div>
            
            <div className="md:hidden flex overflow-x-auto gap-2 px-4 py-2 bg-slate-950 border-t border-cyan-500/20">
              <button onClick={() => setActiveTab('ranking')} className={`px-3 py-1.5 rounded font-montserrat font-bold text-[10px] uppercase tracking-wider whitespace-nowrap ${activeTab === 'ranking' ? 'bg-cyan-950 text-cyan-300 border border-cyan-400' : 'text-cyan-600'}`}>Ranking</button>
              <button onClick={() => setActiveTab('quadro')} className={`px-3 py-1.5 rounded font-montserrat font-bold text-[10px] uppercase tracking-wider whitespace-nowrap ${activeTab === 'quadro' ? 'bg-cyan-950 text-cyan-300 border border-cyan-400' : 'text-cyan-600'}`}>Quadro</button>
              <button onClick={() => setActiveTab('tribos')} className={`px-3 py-1.5 rounded font-montserrat font-bold text-[10px] uppercase tracking-wider whitespace-nowrap ${activeTab === 'tribos' ? 'bg-cyan-950 text-cyan-300 border border-cyan-400' : 'text-cyan-600'}`}>Tribos</button>
              {currentUser.role === 'admin' && <button onClick={() => setActiveTab('timers')} className={`px-3 py-1.5 rounded font-montserrat font-bold text-[10px] uppercase tracking-wider whitespace-nowrap ${activeTab === 'timers' ? 'bg-cyan-950 text-cyan-300 border border-cyan-400' : 'text-cyan-600'}`}>Relógios</button>}
              {currentUser.role === 'admin' && <button onClick={() => setActiveTab('admin')} className={`px-3 py-1.5 rounded font-montserrat font-bold text-[10px] uppercase tracking-wider whitespace-nowrap ${activeTab === 'admin' ? 'bg-cyan-950 text-cyan-300 border border-cyan-400' : 'text-cyan-600'}`}>Provas</button>}
            </div>
          </header>

          <main className="max-w-6xl mx-auto px-4 sm:px-6 lg:px-8 py-10 relative z-10 flex-1 w-full">
            {activeTab === 'ranking' && <RankingTab teamScores={teamScores} completedTasks={completedTasks} teams={teams} currentUser={currentUser} settings={settings} />}
            {activeTab === 'quadro' && <QuestBoardTab pendingTasks={pendingTasks} settings={settings} />}
            {activeTab === 'tribos' && <TribosTab teamScores={teamScores} allTasks={tasks} onUpdateTeam={handleUpdateTeam} currentUser={currentUser} settings={settings} />}
            {activeTab === 'timers' && currentUser.role === 'admin' && <TimersTab timers={timers} toggleTimer={toggleTimer} resetTimer={resetTimer} toggleAllTimers={toggleAllTimers} resetAllTimers={resetAllTimers} teams={teams} currentUser={currentUser} />}
            {activeTab === 'admin' && currentUser.role === 'admin' && <AdminTab pendingTasks={pendingTasks} completedTasks={completedTasks} onCreateTask={handleCreateTask} onDeleteTask={handleDeleteTask} onCompleteTask={handleCompleteTask} showToast={showToast} onLogout={handleLogout} teams={teams} settings={settings} onToggleVisibility={handleToggleVisibility} onUpdateMessage={handleUpdateMessage} penalties={penalties} onAddPenalty={handleAddPenalty} onDeletePenalty={handleDeletePenalty} />}
          </main>
        </div>
      )}

      <ToastNotification toast={toast} />
      
      <style dangerouslySetInnerHTML={{__html: `
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600&family=Montserrat:wght@400;600;700;800&family=JetBrains+Mono:wght@400;700&display=swap');
        body { font-family: 'Inter', sans-serif; }
        .font-montserrat { font-family: 'Montserrat', sans-serif; }
        .font-mono { font-family: 'JetBrains Mono', monospace; }
        .solo-panel { background: rgba(2, 6, 23, 0.85); backdrop-filter: blur(16px); -webkit-backdrop-filter: blur(16px); border: 1px solid rgba(6, 182, 212, 0.35); box-shadow: 0 0 25px rgba(6, 182, 212, 0.1), inset 0 0 15px rgba(6, 182, 212, 0.05); }
        @keyframes lightning-flash { 0%, 92%, 96% { opacity: 0; } 93%, 97% { opacity: 0.2; background: radial-gradient(circle at 50% -20%, rgba(6, 182, 212, 0.5) 0%, transparent 60%); } 100% { opacity: 0; } }
        .lightning-overlay { animation: lightning-flash 7s infinite; }
        @keyframes fade-in { from { opacity: 0; } to { opacity: 1; } }
        @keyframes fade-in-up { from { opacity: 0; transform: translateY(15px); } to { opacity: 1; transform: translateY(0); } }
        .animate-fade-in { animation: fade-in 0.4s ease-out; }
        .animate-fade-in-up { animation: fade-in-up 0.4s ease-out forwards; }
      `}} />
    </div>
  );
}
