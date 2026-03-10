```jsx
import React, { useState, useEffect, useMemo, useRef } from 'react';
import { 
  LineChart, Line, BarChart, Bar, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer, PieChart, Pie, Cell, AreaChart, Area, Legend
} from 'recharts';
import { 
  LayoutDashboard, MessageSquare, Database, Settings, Activity, 
  AlertCircle, CheckCircle, Search, ArrowUpRight, Upload, X, ChevronDown,
  Server, HardDrive, Key, FileJson, Lock, User, Bot, Terminal, 
  ChevronLeft, ChevronRight, Sliders, Filter, Clock, Users, GitBranch, Zap, Flag, Send, Tag, Code, Eye, Sun, Moon, List, MousePointerClick, TrendingUp, UserMinus, ShieldCheck, Share2, BarChart2
} from 'lucide-react';
import './index.css'

// --- UTILS: STORAGE ENGINE ---
const STORAGE_KEY_DATA = 'rasalens_db_v9';
const STORAGE_KEY_SETTINGS = 'rasalens_settings_v9';

const db = {
  save: (data) => {
    try {
      localStorage.setItem(STORAGE_KEY_DATA, JSON.stringify(data));
    } catch (e) {
      console.error("Storage quota exceeded", e);
    }
  },
  load: () => {
    try {
      const data = localStorage.getItem(STORAGE_KEY_DATA);
      return data ? JSON.parse(data) : null;
    } catch (e) {
      return null;
    }
  },
  clear: () => {
    localStorage.removeItem(STORAGE_KEY_DATA);
  },
  saveSettings: (settings) => {
    localStorage.setItem(STORAGE_KEY_SETTINGS, JSON.stringify(settings));
  },
  loadSettings: () => {
    try {
      const s = localStorage.getItem(STORAGE_KEY_SETTINGS);
      return s ? JSON.parse(s) : null;
    } catch (e) {
      return null;
    }
  }
};

// --- UTILS: DATA PROCESSING ENGINE ---

const processData = (rawData) => {
  if (!rawData || !Array.isArray(rawData)) return null;

  let totalEvents = 0;
  let totalConfidence = 0;
  let intentCounts = {};
  let actionCounts = {};
  let entityCounts = {};
  let flowCounts = {};
  let channelCounts = {};
  let fallbackCount = 0;
  let escalationCount = 0;
  let abandonmentCount = 0;
  
  // Time buckets
  let sessionsPerDate = {};
  // Session complexity for the new chart
  let sessionComplexity = [];

  let conversations = [];
  
  // Latency & Stats
  let latencySum = 0;
  let latencyCount = 0;
  let latenciesOverTime = []; 
  // Enhanced session lengths tracking
  let sessionLengths = { 
    short: { count: 0, events: 0 }, 
    medium: { count: 0, events: 0 }, 
    long: { count: 0, events: 0 } 
  };
  let totalTurns = 0;

  // Flow Stats (CALM)
  let flowStats = {}; 
  let totalFlows = 0;
  let completedFlows = 0;

  rawData.forEach((tracker) => {
    const events = tracker.events || [];
    let sessionConfidenceSum = 0;
    let sessionIntentCount = 0;
    let lastUserTimestamp = null;
    let hasEscalation = false;
    let sessionChannel = 'unknown';
    
    // For Session Complexity Chart
    sessionComplexity.push({
        id: tracker.sender_id,
        eventCount: events.length,
        turnCount: events.filter(e => e.event === 'user').length
    });
    
    // --- ENRICHED HISTORY GENERATION ---
    let enrichedHistory = [];
    let currentTurn = null;

    events.forEach(e => {
        totalEvents++;

        // 1. User Event - Start of a Turn
        if (e.event === 'user') {
            lastUserTimestamp = e.timestamp;
            totalTurns++;
            
            // Channel Detection
            if (e.input_channel) {
                sessionChannel = e.input_channel;
            }

            // Stats
            if (e.parse_data?.intent) {
                const intent = e.parse_data.intent.name;
                const conf = e.parse_data.intent.confidence || 0;
                
                if (!intentCounts[intent]) intentCounts[intent] = { count: 0, confSum: 0 };
                intentCounts[intent].count++;
                intentCounts[intent].confSum += conf;
                
                totalConfidence += conf;
                sessionConfidenceSum += conf;
                sessionIntentCount++;

                if (intent === 'nlu_fallback') fallbackCount++;
                
                // Escalation Heuristic: specific intents
                if (['human_handoff', 'request_human', 'agent', 'escalate'].includes(intent)) {
                    hasEscalation = true;
                }
            }

            // Entity Extraction
            if (e.parse_data?.entities) {
                e.parse_data.entities.forEach(ent => {
                    if (!entityCounts[ent.entity]) entityCounts[ent.entity] = { count: 0, values: new Set() };
                    entityCounts[ent.entity].count++;
                    if (ent.value) entityCounts[ent.entity].values.add(ent.value);
                });
            }

            // Command / Flow Trigger Extraction (CALM)
            let triggeredFlow = null;
            if (e.parse_data?.commands) {
                e.parse_data.commands.forEach(cmd => {
                    if (cmd.command === 'start flow' && cmd.flow) {
                        triggeredFlow = cmd.flow;
                        flowCounts[cmd.flow] = (flowCounts[cmd.flow] || 0) + 1;
                    }
                });
            }

            // Finalize previous turn
            if (currentTurn) enrichedHistory.push(currentTurn);

            // Start new turn
            currentTurn = {
                role: 'user',
                text: e.text,
                timestamp: e.timestamp,
                intent: e.parse_data?.intent?.name,
                conf: e.parse_data?.intent?.confidence,
                entities: e.parse_data?.entities || [],
                triggered_flow: triggeredFlow,
                slots_set: [],
                events_after: [],
                simulated: false
            };
        }
        
        // 2. Events following user message (Consequences)
        else if (currentTurn) {
             if (e.event === 'bot') {
                 enrichedHistory.push(currentTurn);
                 currentTurn = null; 
                 
                 // Latency calc
                 if (lastUserTimestamp) {
                    const diff = (e.timestamp - lastUserTimestamp) * 1000; 
                    if (diff > 0 && diff < 60000) { 
                        latencySum += diff;
                        latencyCount++;
                        // Safe date parsing
                        if (e.timestamp && !isNaN(e.timestamp)) {
                            const date = new Date(e.timestamp * 1000);
                            const timeKey = `${date.getHours()}:00`;
                            latenciesOverTime.push({ time: timeKey, ms: diff });
                        }
                    }
                    lastUserTimestamp = null; 
                 }
                 
                 enrichedHistory.push({
                     role: 'bot',
                     text: e.text,
                     timestamp: e.timestamp,
                     simulated: false,
                     metadata: e.metadata
                 });

             } else {
                 if (currentTurn) {
                     if (e.event === 'slot') {
                         currentTurn.slots_set.push({ name: e.name, value: e.value });
                     }
                     if (e.event === 'flow_started') {
                         currentTurn.active_flow = e.flow_id; 
                     }
                     currentTurn.events_after.push(e);
                 }
             }
        }

        // 3. Independent Stats Aggregation
        if (e.event === 'action') {
             if (e.name) actionCounts[e.name] = (actionCounts[e.name] || 0) + 1;
             // Heuristic for escalation via action
             if (e.name === 'action_human_handoff' || e.name === 'action_transfer') hasEscalation = true;
        }
        if (e.event === 'flow_started') {
             const flowId = e.flow_id;
             if (flowId) {
                if (!flowStats[flowId]) flowStats[flowId] = { id: flowId, triggered: 0, completed: 0 };
                flowStats[flowId].triggered++;
                totalFlows++;
             }
        }
        if (e.event === 'flow_completed') {
            const flowId = e.flow_id;
            if (flowId && flowStats[flowId]) {
                flowStats[flowId].completed++;
                completedFlows++;
            }
        }
    });
    if (currentTurn) enrichedHistory.push(currentTurn);

    // --- SESSION LEVEL METRICS ---
    
    // 1. Channel Counting
    channelCounts[sessionChannel] = (channelCounts[sessionChannel] || 0) + 1;

    // 2. Escalation
    if (hasEscalation) escalationCount++;

    // 3. Abandonment Logic (Ended while in a flow or loop)
    // Checking tracker stack for active flow frame
    const activeFlowFrame = tracker.stack && tracker.stack.find(f => f.type === 'flow');
    if (activeFlowFrame) {
        abandonmentCount++;
    }

    // 4. Time Bucketing
    const lastTimestampVal = events[events.length - 1]?.timestamp;
    if (lastTimestampVal && !isNaN(lastTimestampVal)) {
        const dateObj = new Date(lastTimestampVal * 1000);
        const dateKey = dateObj.toLocaleDateString('en-CA'); // YYYY-MM-DD
        sessionsPerDate[dateKey] = (sessionsPerDate[dateKey] || 0) + 1;
    }

    const lastTimestamp = (lastTimestampVal && !isNaN(lastTimestampVal))
      ? new Date(lastTimestampVal * 1000).toLocaleString() 
      : 'Unknown';

    // Length Bucket
    const turnCount = events.filter(e => e.event === 'user').length;
    const evtCount = events.length;

    if (turnCount < 5) {
        sessionLengths.short.count++;
        sessionLengths.short.events += evtCount;
    } else if (turnCount <= 20) {
        sessionLengths.medium.count++;
        sessionLengths.medium.events += evtCount;
    } else {
        sessionLengths.long.count++;
        sessionLengths.long.events += evtCount;
    }

    const latestUserEvent = events.filter(e => e.event === 'user').pop();
    const latestIntent = latestUserEvent?.parse_data?.intent?.name || 'unknown';
    const latestConf = latestUserEvent?.parse_data?.intent?.confidence || 0;
    const activeFlow = tracker.stack && tracker.stack.find(f => f.type === 'flow')?.flow_id;

    conversations.push({
      id: tracker.sender_id,
      latest_intent: latestIntent,
      active_flow: activeFlow,
      confidence: latestConf,
      timestamp: lastTimestamp,
      timestampRaw: lastTimestampVal,
      status: latestIntent === 'nlu_fallback' || latestConf < 0.6 ? 'review' : 'active',
      history: enrichedHistory,
      raw_events: events,
      avg_confidence: sessionIntentCount > 0 ? (sessionConfidenceSum / sessionIntentCount) : 0,
      message_count: enrichedHistory.length,
      flagged: tracker.flagged || false,
      channel: sessionChannel,
      _processed: true 
    });
  });

  // --- AGGREGATION FOR CHARTS ---

  const intentChartData = Object.keys(intentCounts)
    .map(key => ({ name: key, value: intentCounts[key].count, avgConf: intentCounts[key].confSum / intentCounts[key].count }))
    .sort((a, b) => b.value - a.value);

  const flowChartData = Object.keys(flowCounts)
    .map(key => ({ name: key, value: flowCounts[key] }))
    .sort((a, b) => b.value - a.value);

  const primaryDistribution = totalFlows > (totalEvents * 0.05) ? 'flows' : 'intents';

  const actionChartData = Object.keys(actionCounts)
    .map(key => ({ name: key, value: actionCounts[key] }))
    .sort((a, b) => b.value - a.value)
    .slice(0, 10);

  const entityChartData = Object.keys(entityCounts)
    .map(key => ({ name: key, value: entityCounts[key].count, examples: Array.from(entityCounts[key].values).slice(0,5) }))
    .sort((a, b) => b.value - a.value);

  const channelChartData = Object.keys(channelCounts).map(key => ({ name: key, value: channelCounts[key] }));

  const sessionsOverTimeData = Object.keys(sessionsPerDate).map(date => ({
      date,
      count: sessionsPerDate[date]
  })).sort((a, b) => a.date.localeCompare(b.date));

  // Sort by event count descending for the complexity chart
  const sessionComplexityData = sessionComplexity
      .sort((a, b) => b.eventCount - a.eventCount)
      .slice(0, 50); // Top 50

  const latencyMap = {};
  latenciesOverTime.forEach(l => {
      if (!latencyMap[l.time]) latencyMap[l.time] = { sum: 0, count: 0 };
      latencyMap[l.time].sum += l.ms;
      latencyMap[l.time].count++;
  });
  const latencyChartData = Object.keys(latencyMap).map(time => ({
      time,
      ms: Math.round(latencyMap[time].sum / latencyMap[time].count)
  })).sort((a,b) => parseInt(a.time) - parseInt(b.time));

  const flowAnalytics = Object.values(flowStats)
    .map(f => ({
        name: f.id,
        triggered: f.triggered,
        completed: f.completed,
        dropoff: f.triggered - f.completed,
        rate: f.triggered > 0 ? (f.completed / f.triggered) : 0
    }))
    .sort((a, b) => b.triggered - a.triggered);

  const sessionLengthData = [
      { 
          name: 'Short (<5 turns)', 
          value: sessionLengths.short.count, 
          avgEvents: Math.round(sessionLengths.short.events / (sessionLengths.short.count || 1)) 
      },
      { 
          name: 'Medium (5-20 turns)', 
          value: sessionLengths.medium.count, 
          avgEvents: Math.round(sessionLengths.medium.events / (sessionLengths.medium.count || 1)) 
      },
      { 
          name: 'Long (20+ turns)', 
          value: sessionLengths.long.count, 
          avgEvents: Math.round(sessionLengths.long.events / (sessionLengths.long.count || 1)) 
      }
  ];

  return {
    metrics: {
      totalConversations: rawData.length,
      avgConfidence: totalEvents > 0 ? (totalConfidence / totalEvents) : 0,
      fallbackRate: totalEvents > 0 ? (fallbackCount / totalEvents) : 0,
      activeUsers: rawData.length,
      totalFlowsTriggered: totalFlows,
      flowCompletionRate: totalFlows > 0 ? (completedFlows / totalFlows) : 0,
      avgLatency: latencyCount > 0 ? (latencySum / latencyCount) : 0,
      totalEvents: totalEvents,
      avgTurns: totalTurns > 0 ? Math.round(totalTurns / rawData.length) : 0,
      primaryDistribution,
      escalationRate: rawData.length > 0 ? (escalationCount / rawData.length) : 0,
      containmentRate: rawData.length > 0 ? (1 - (escalationCount / rawData.length)) : 0,
      abandonmentRate: rawData.length > 0 ? (abandonmentCount / rawData.length) : 0
    },
    charts: {
        intents: intentChartData,
        flows: flowChartData,
        actions: actionChartData,
        entities: entityChartData,
        latency: latencyChartData,
        sessionLengths: sessionLengthData,
        channels: channelChartData,
        sessionsOverTime: sessionsOverTimeData,
        sessionComplexity: sessionComplexityData
    },
    flows: flowAnalytics,
    conversations: conversations.sort((a, b) => b.timestamp.localeCompare(a.timestamp)),
    _raw: rawData 
  };
};

// --- UI COMPONENTS ---
const Card = ({ children, className = "", onClick }) => (
  <div 
    onClick={onClick}
    className={`bg-white dark:bg-slate-800 rounded-xl border border-slate-200 dark:border-slate-700 shadow-sm transition-all duration-200 ${className} ${onClick ? 'cursor-pointer hover:border-slate-300 dark:hover:border-slate-600 hover:shadow-md active:scale-[0.99]' : ''}`}
  >
    {children}
  </div>
);

const Button = ({ children, variant = "primary", className = "", loading, onClick, ...props }) => {
  const baseStyles = "inline-flex items-center rounded-md text-sm font-medium transition-all focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-slate-950 disabled:pointer-events-none disabled:opacity-50 h-9 px-4 py-2";
  const variants = {
    primary: "bg-slate-900 text-slate-50 hover:bg-slate-800 dark:bg-slate-50 dark:text-slate-900 dark:hover:bg-slate-200 shadow hover:shadow-md justify-center",
    secondary: "bg-slate-100 text-slate-900 hover:bg-slate-200 dark:bg-slate-800 dark:text-slate-50 dark:hover:bg-slate-700 justify-center",
    outline: "border border-slate-200 bg-transparent shadow-sm hover:bg-slate-50 hover:text-slate-900 dark:border-slate-700 dark:text-slate-300 dark:hover:bg-slate-800 dark:hover:text-slate-50 justify-center",
    ghost: "hover:bg-slate-100 hover:text-slate-900 text-slate-600 dark:text-slate-400 dark:hover:bg-slate-800 dark:hover:text-slate-50",
    danger: "bg-red-500 text-white hover:bg-red-600 justify-center",
    dangerOutline: "border border-red-200 text-red-600 bg-red-50 hover:bg-red-100 dark:border-red-900 dark:bg-red-900/20 dark:text-red-400 justify-center",
    emerald: "bg-emerald-600 text-white hover:bg-emerald-700 shadow-sm hover:shadow-md justify-center"
  };
  return (
    <button className={`${baseStyles} ${variants[variant]} ${className}`} onClick={onClick} disabled={loading} {...props}>
      {loading ? (
        <span className="mr-2 h-4 w-4 animate-spin rounded-full border-2 border-current border-t-transparent" />
      ) : null}
      {children}
    </button>
  );
};

const Input = ({ className = "", label, ...props }) => (
  <div className="space-y-2 w-full">
    {label && <label className="text-sm font-medium leading-none text-slate-700 dark:text-slate-300">{label}</label>}
    <input 
      className={`flex h-9 w-full rounded-md border border-slate-200 dark:border-slate-700 bg-transparent px-3 py-1 text-sm shadow-sm transition-colors file:border-0 file:bg-transparent file:text-sm file:font-medium placeholder:text-slate-400 focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-slate-950 disabled:cursor-not-allowed disabled:opacity-50 dark:text-slate-100 ${className}`}
      {...props}
    />
  </div>
);

const Badge = ({ children, variant = "default", className = "" }) => {
  const variants = {
    default: "border-transparent bg-slate-900 text-slate-50 dark:bg-slate-50 dark:text-slate-900",
    secondary: "border-transparent bg-slate-100 text-slate-900 dark:bg-slate-800 dark:text-slate-100",
    outline: "text-slate-600 border border-slate-200 dark:text-slate-400 dark:border-slate-700",
    destructive: "border-transparent bg-red-50 text-red-600 border border-red-100 dark:bg-red-900/30 dark:text-red-400 dark:border-red-900/50",
    success: "border-transparent bg-emerald-50 text-emerald-600 border border-emerald-100 dark:bg-emerald-900/30 dark:text-emerald-400 dark:border-emerald-900/50",
    blue: "border-transparent bg-blue-50 text-blue-600 border border-blue-100 dark:bg-blue-900/30 dark:text-blue-400 dark:border-blue-900/50",
    indigo: "border-transparent bg-indigo-50 text-indigo-600 border border-indigo-100 dark:bg-indigo-900/30 dark:text-indigo-400 dark:border-indigo-900/50",
    purple: "border-transparent bg-purple-50 text-purple-700 border border-purple-200 dark:bg-purple-900/30 dark:text-purple-400 dark:border-purple-900/50",
    teal: "border-transparent bg-teal-50 text-teal-700 border border-teal-200 dark:bg-teal-900/30 dark:text-teal-400 dark:border-teal-900/50"
  };
  return (
    <div className={`inline-flex items-center rounded-md border px-2.5 py-0.5 text-xs font-semibold transition-colors focus:outline-none focus:ring-2 focus:ring-slate-950 focus:ring-offset-2 ${variants[variant]} ${className}`}>
      {children}
    </div>
  );
};

const Modal = ({ isOpen, onClose, title, children }) => {
  if (!isOpen) return null;
  return (
    <div className="fixed inset-0 z-50 flex items-center justify-center p-4">
      <div className="fixed inset-0 bg-slate-900/40 backdrop-blur-sm" onClick={onClose} />
      <div className="z-50 w-full max-w-4xl bg-white dark:bg-slate-900 rounded-xl shadow-2xl animate-in zoom-in-95 duration-200 overflow-hidden flex flex-col max-h-[85vh] border border-slate-200 dark:border-slate-800">
        <div className="flex items-center justify-between p-6 border-b border-slate-100 dark:border-slate-800">
          <h2 className="text-lg font-semibold text-slate-900 dark:text-slate-100">{title}</h2>
          <button onClick={onClose} className="text-slate-400 hover:text-slate-600 dark:hover:text-slate-200 transition-colors">
            <X className="h-5 w-5" />
          </button>
        </div>
        <div className="flex-1 overflow-auto p-0 dark:text-slate-300">
          {children}
        </div>
      </div>
    </div>
  );
};

const UserTableModal = ({ isOpen, onClose, users }) => {
  const [sortConfig, setSortConfig] = useState({ key: 'timestamp', direction: 'desc' });
  
  const sortedUsers = useMemo(() => {
    let sortable = [...(users || [])];
    sortable.sort((a, b) => {
      let aVal = a[sortConfig.key];
      let bVal = b[sortConfig.key];
      
      // Handle numeric vs string
      if (typeof aVal === 'string') aVal = aVal.toLowerCase();
      if (typeof bVal === 'string') bVal = bVal.toLowerCase();

      if (aVal < bVal) return sortConfig.direction === 'asc' ? -1 : 1;
      if (aVal > bVal) return sortConfig.direction === 'asc' ? 1 : -1;
      return 0;
    });
    return sortable;
  }, [users, sortConfig]);

  const requestSort = (key) => {
    let direction = 'asc';
    if (sortConfig.key === key && sortConfig.direction === 'asc') {
      direction = 'desc';
    }
    setSortConfig({ key, direction });
  };

  const getSortIcon = (key) => {
    if (sortConfig.key !== key) return <div className="w-3 h-3 ml-1 inline-block opacity-20">↕</div>;
    return sortConfig.direction === 'asc' ? <span className="ml-1 inline-block">↑</span> : <span className="ml-1 inline-block">↓</span>;
  };

  return (
    <Modal isOpen={isOpen} onClose={onClose} title={`Unique Users List (${users?.length || 0})`}>
       <div className="overflow-x-auto">
        <table className="w-full text-sm text-left text-slate-500 dark:text-slate-400">
            <thead className="text-xs text-slate-700 uppercase bg-slate-50 dark:bg-slate-800 dark:text-slate-400 sticky top-0 z-10">
                <tr>
                    <th className="px-6 py-3 cursor-pointer hover:bg-slate-100 dark:hover:bg-slate-700 transition-colors" onClick={() => requestSort('id')}>
                        <div className="flex items-center">User ID {getSortIcon('id')}</div>
                    </th>
                    <th className="px-6 py-3 cursor-pointer hover:bg-slate-100 dark:hover:bg-slate-700 transition-colors" onClick={() => requestSort('timestamp')}>
                        <div className="flex items-center">Last Seen {getSortIcon('timestamp')}</div>
                    </th>
                    <th className="px-6 py-3 cursor-pointer hover:bg-slate-100 dark:hover:bg-slate-700 transition-colors" onClick={() => requestSort('message_count')}>
                         <div className="flex items-center">Events {getSortIcon('message_count')}</div>
                    </th>
                    <th className="px-6 py-3 cursor-pointer hover:bg-slate-100 dark:hover:bg-slate-700 transition-colors" onClick={() => requestSort('channel')}>
                         <div className="flex items-center">Channel {getSortIcon('channel')}</div>
                    </th>
                    <th className="px-6 py-3">Latest Status</th>
                </tr>
            </thead>
            <tbody className="divide-y divide-slate-100 dark:divide-slate-800">
                {sortedUsers.map((user, i) => (
                    <tr key={i} className="bg-white dark:bg-slate-900 hover:bg-slate-50 dark:hover:bg-slate-800/50 transition-colors">
                        <td className="px-6 py-4 font-medium text-slate-900 dark:text-white">{user.id}</td>
                        <td className="px-6 py-4 whitespace-nowrap">{user.timestamp}</td>
                        <td className="px-6 py-4">{user.message_count}</td>
                         <td className="px-6 py-4">
                             {user.channel !== 'unknown' ? <Badge variant="outline">{user.channel}</Badge> : <span className="text-slate-300">-</span>}
                        </td>
                        <td className="px-6 py-4">
                            <div className="flex items-center gap-2">
                                {user.flagged && <Flag className="h-3 w-3 text-red-500" />}
                                <span className={`truncate max-w-[150px] ${user.status === 'review' ? 'text-amber-600' : ''}`}>{user.latest_intent}</span>
                            </div>
                        </td>
                    </tr>
                ))}
            </tbody>
        </table>
       </div>
    </Modal>
  );
};

// --- SCREENS ---

const ConnectionGateway = ({ onConnect }) => {
  const [loading, setLoading] = useState(false);
  const fileInputRef = useRef(null);
  const [fileName, setFileName] = useState(null);
  const [storeType, setStoreType] = useState('json');

  const handleFileChange = (event) => {
    const file = event.target.files?.[0];
    if (file) {
      setFileName(file.name);
      const reader = new FileReader();
      reader.onload = (e) => {
        try {
          const json = JSON.parse(e.target.result);
          onConnect('json', json);
        } catch (error) {
          alert("Error parsing JSON file. Please ensure it is a valid Rasa tracker dump.");
        }
      };
      reader.readAsText(file);
    }
  };

  return (
    <div className="min-h-screen bg-slate-50 dark:bg-slate-950 flex flex-col items-center justify-center p-4">
      <div className="mb-8 text-center space-y-2">
         <div className="mx-auto w-12 h-12 bg-slate-900 dark:bg-slate-50 rounded-xl flex items-center justify-center text-white dark:text-slate-900 mb-4 shadow-xl shadow-slate-200 dark:shadow-none">
          <span className="font-bold text-xl">R</span>
        </div>
        <h1 className="text-3xl font-bold text-slate-900 dark:text-slate-50 tracking-tight">RasaLens Analytics</h1>
        <p className="text-slate-500 dark:text-slate-400 max-w-md mx-auto">Upload your Rasa tracker store JSON dump to visualize conversations, flows, and NLU performance.</p>
      </div>

      <Card className="w-full max-w-xl overflow-hidden shadow-2xl shadow-slate-200/50 dark:shadow-none border-0 ring-1 ring-slate-200 dark:ring-slate-800 p-0">
         <div className="flex border-b border-slate-100 dark:border-slate-800">
            <button onClick={() => setStoreType('json')} className={`flex-1 py-4 text-sm font-medium ${storeType === 'json' ? 'bg-slate-50 dark:bg-slate-800 text-slate-900 dark:text-slate-100' : 'text-slate-400'}`}>File Upload</button>
            <button onClick={() => setStoreType('mongo')} className={`flex-1 py-4 text-sm font-medium ${storeType === 'mongo' ? 'bg-slate-50 dark:bg-slate-800 text-slate-900 dark:text-slate-100' : 'text-slate-400'}`}>Mongo DB</button>
            <button onClick={() => setStoreType('sql')} className={`flex-1 py-4 text-sm font-medium ${storeType === 'sql' ? 'bg-slate-50 dark:bg-slate-800 text-slate-900 dark:text-slate-100' : 'text-slate-400'}`}>SQL</button>
         </div>
         <div className="p-8">
            {storeType === 'json' && (
                <div 
                    onClick={() => fileInputRef.current?.click()}
                    className="h-[200px] border-2 border-dashed border-slate-200 dark:border-slate-700 rounded-lg flex flex-col items-center justify-center text-center hover:bg-slate-50 dark:hover:bg-slate-800/50 transition cursor-pointer group"
                >
                    <input type="file" ref={fileInputRef} className="hidden" accept=".json" onChange={handleFileChange} />
                    <div className="flex flex-col items-center">
                        <div className="p-3 bg-indigo-50 dark:bg-indigo-900/20 rounded-full mb-3 text-indigo-500 group-hover:bg-indigo-100 dark:group-hover:bg-indigo-900/40 group-hover:text-indigo-600 transition-colors">
                            <Upload className="h-8 w-8" />
                        </div>
                        <p className="font-medium text-slate-900 dark:text-slate-200">Upload tracker_dump.json</p>
                        <p className="text-xs text-slate-400 mt-2">Supports Rasa 3.x / CALM</p>
                    </div>
                </div>
            )}
            {storeType !== 'json' && (
                <div className="h-[200px] flex flex-col items-center justify-center text-center">
                   <HardDrive className="h-10 w-10 text-slate-300 mb-4" />
                   <p className="text-slate-500 max-w-sm">Direct database connection is disabled in the browser demo. Please use the JSON upload for the MVP.</p>
               </div>
            )}
         </div>
      </Card>
      
      <p className="mt-8 text-xs text-slate-400">Data is processed locally in your browser.</p>
    </div>
  );
};

const MetricCard = ({ title, value, sub, icon: Icon, trend, trendColor, onClick }) => (
  <Card onClick={onClick} className="p-6 relative overflow-hidden group hover:shadow-lg transition-shadow">
    <div className="flex items-center justify-between pb-2 z-10 relative">
      <h3 className="text-sm font-medium text-slate-600 dark:text-slate-400 group-hover:text-slate-900 dark:group-hover:text-slate-200 transition-colors">{title}</h3>
      <Icon className="h-4 w-4 text-slate-400 group-hover:text-slate-600 dark:group-hover:text-slate-300 transition-colors" />
    </div>
    <div className="text-2xl font-bold text-slate-900 dark:text-slate-100 z-10 relative">{value}</div>
    <p className={`text-xs mt-1 flex items-center font-medium ${trendColor || 'text-slate-500 dark:text-slate-400'}`}>
      {trend === 'up' && <ArrowUpRight className="h-3 w-3 mr-1" />}
      {sub}
    </p>
  </Card>
);

const CustomPieTooltip = ({ active, payload }) => {
  if (active && payload && payload.length) {
    const data = payload[0].payload;
    return (
      <div className="bg-white dark:bg-slate-800 p-3 border border-slate-200 dark:border-slate-700 rounded-lg shadow-lg z-50">
        <p className="font-semibold text-slate-900 dark:text-slate-100">{data.name}</p>
        <p className="text-sm text-slate-600 dark:text-slate-400">Sessions: <span className="font-medium text-slate-900 dark:text-slate-200">{data.value}</span></p>
        <p className="text-sm text-slate-600 dark:text-slate-400">Avg Events: <span className="font-medium text-slate-900 dark:text-slate-200">{data.avgEvents}</span></p>
      </div>
    );
  }
  return null;
};

const OverviewTab = ({ data, onDrillDown }) => {
  const [distributionMode, setDistributionMode] = useState(data.metrics.primaryDistribution); 
  const [showUsersModal, setShowUsersModal] = useState(false);

  return (
  <div className="space-y-6 animate-in fade-in duration-500">
    <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-4">
      <MetricCard 
        title="Unique Users" 
        value={data?.metrics?.activeUsers || 0} 
        sub={`${data?.metrics?.totalConversations} active sessions`} 
        icon={Users} 
        trend="up" 
        trendColor="text-emerald-600"
        onClick={() => setShowUsersModal(true)}
      />
      <MetricCard 
        title="Avg. Confidence" 
        value={`${((data?.metrics?.avgConfidence || 0) * 100).toFixed(1)}%`}
        sub="Global average" 
        icon={CheckCircle}
        onClick={() => onDrillDown('avg_confidence')}
      />
      <MetricCard 
        title="Flow Completion" 
        value={`${((data?.metrics?.flowCompletionRate || 0) * 100).toFixed(0)}%`}
        sub={`${data?.metrics?.totalFlowsTriggered} flows triggered`}
        icon={GitBranch}
        trendColor="text-blue-600"
        onClick={() => onDrillDown('performance')}
      />
      <MetricCard 
        title="Avg Session Length" 
        value={`${data?.metrics?.avgTurns || 0} turns`}
        sub="Engagement depth" 
        icon={Clock}
        onClick={() => onDrillDown('total_conversations')}
      />
    </div>

    {/* BUSINESS METRICS ROW */}
    <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
        <Card className="p-5 flex items-center gap-4 border-l-4 border-l-emerald-500">
             <div className="p-3 bg-emerald-50 dark:bg-emerald-900/20 rounded-full text-emerald-600 dark:text-emerald-400">
                 <ShieldCheck className="h-6 w-6" />
             </div>
             <div>
                 <p className="text-xs text-slate-500 dark:text-slate-400 font-medium uppercase tracking-wider">Containment Rate</p>
                 <div className="text-2xl font-bold text-slate-900 dark:text-slate-100">{((data?.metrics?.containmentRate || 0)*100).toFixed(1)}%</div>
             </div>
        </Card>
        <Card className="p-5 flex items-center gap-4 border-l-4 border-l-amber-500">
             <div className="p-3 bg-amber-50 dark:bg-amber-900/20 rounded-full text-amber-600 dark:text-amber-400">
                 <UserMinus className="h-6 w-6" />
             </div>
             <div>
                 <p className="text-xs text-slate-500 dark:text-slate-400 font-medium uppercase tracking-wider">Abandonment Rate</p>
                 <div className="text-2xl font-bold text-slate-900 dark:text-slate-100">{((data?.metrics?.abandonmentRate || 0)*100).toFixed(1)}%</div>
             </div>
        </Card>
        <Card className="p-5 flex items-center gap-4 border-l-4 border-l-red-500">
             <div className="p-3 bg-red-50 dark:bg-red-900/20 rounded-full text-red-600 dark:text-red-400">
                 <Share2 className="h-6 w-6" />
             </div>
             <div>
                 <p className="text-xs text-slate-500 dark:text-slate-400 font-medium uppercase tracking-wider">Escalation Rate</p>
                 <div className="text-2xl font-bold text-slate-900 dark:text-slate-100">{((data?.metrics?.escalationRate || 0)*100).toFixed(1)}%</div>
             </div>
        </Card>
    </div>

    <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
      <Card className="lg:col-span-2 p-6">
        <div className="mb-6 flex justify-between items-center">
            <div>
                <h3 className="font-semibold text-slate-900 dark:text-slate-100">{distributionMode === 'flows' ? 'Top Triggered Flows' : 'Intent Distribution'}</h3>
                <p className="text-sm text-slate-500 dark:text-slate-400">{distributionMode === 'flows' ? 'Most popular business logic flows (CALM).' : 'Most frequently matched NLU intents.'}</p>
            </div>
            <div className="flex bg-slate-100 dark:bg-slate-700 rounded-lg p-0.5">
                <button 
                    onClick={() => setDistributionMode('intents')}
                    className={`px-3 py-1 text-xs font-medium rounded-md transition-all ${distributionMode === 'intents' ? 'bg-white dark:bg-slate-600 text-slate-900 dark:text-slate-100 shadow-sm' : 'text-slate-500 dark:text-slate-400'}`}
                >Intents</button>
                <button 
                    onClick={() => setDistributionMode('flows')}
                    className={`px-3 py-1 text-xs font-medium rounded-md transition-all ${distributionMode === 'flows' ? 'bg-white dark:bg-slate-600 text-slate-900 dark:text-slate-100 shadow-sm' : 'text-slate-500 dark:text-slate-400'}`}
                >Flows</button>
            </div>
        </div>
        <div className="h-[300px] w-full">
            <ResponsiveContainer width="100%" height="100%">
            <BarChart data={distributionMode === 'flows' ? data?.charts?.flows : data?.charts?.intents} layout="vertical" margin={{ left: 40, right: 20 }}>
                <CartesianGrid strokeDasharray="3 3" horizontal={true} vertical={false} stroke="#f1f5f9" className="dark:opacity-10" />
                <XAxis type="number" hide />
                <YAxis dataKey="name" type="category" width={140} tick={{fontSize: 12, fill: '#64748b'}} />
                <Tooltip 
                    cursor={{fill: 'transparent'}} 
                    contentStyle={{borderRadius: '8px', border: 'none', boxShadow: '0 4px 6px -1px rgb(0 0 0 / 0.1)', backgroundColor: '#fff', color: '#1e293b'}} 
                />
                <Bar dataKey="value" fill={distributionMode === 'flows' ? "#6366f1" : "#3b82f6"} radius={[0, 4, 4, 0]} barSize={20} />
            </BarChart>
            </ResponsiveContainer>
        </div>
      </Card>
      
       <Card className="p-6">
        <div className="mb-6">
          <h3 className="font-semibold text-slate-900 dark:text-slate-100">Session Distribution</h3>
          <p className="text-sm text-slate-500 dark:text-slate-400">Sessions by length (turns).</p>
        </div>
        <div className="h-[300px] w-full relative">
          <ResponsiveContainer width="100%" height="100%">
            <PieChart>
              <Pie
                data={data?.charts?.sessionLengths}
                cx="50%"
                cy="50%"
                innerRadius={60}
                outerRadius={80}
                paddingAngle={5}
                dataKey="value"
                stroke="none"
              >
                {data?.charts?.sessionLengths?.map((entry, index) => (
                  <Cell key={`cell-${index}`} fill={['#10b981', '#3b82f6', '#8b5cf6'][index % 3]} />
                ))}
              </Pie>
              <Tooltip content={<CustomPieTooltip />} />
              <Legend verticalAlign="bottom" height={36}/>
            </PieChart>
          </ResponsiveContainer>
        </div>
      </Card>
    </div>

    {/* ACTIVITY OVER TIME */}
    <Card className="p-6">
        <div className="mb-6">
          <h3 className="font-semibold text-slate-900 dark:text-slate-100">Session Activity Volume</h3>
          <p className="text-sm text-slate-500 dark:text-slate-400">Conversations processed over time.</p>
        </div>
        <div className="h-[250px] w-full">
            {data?.charts?.sessionsOverTime?.length > 0 ? (
                <ResponsiveContainer width="100%" height="100%">
                    <AreaChart data={data.charts.sessionsOverTime}>
                        <defs>
                            <linearGradient id="colorCount" x1="0" y1="0" x2="0" y2="1">
                            <stop offset="5%" stopColor="#3b82f6" stopOpacity={0.8}/>
                            <stop offset="95%" stopColor="#3b82f6" stopOpacity={0}/>
                            </linearGradient>
                        </defs>
                        <CartesianGrid strokeDasharray="3 3" vertical={false} stroke="#f1f5f9" className="dark:opacity-10" />
                        <XAxis dataKey="date" axisLine={false} tickLine={false} tick={{fill: '#64748b', fontSize: 12}} />
                        <YAxis axisLine={false} tickLine={false} tick={{fill: '#64748b', fontSize: 12}} />
                        <Tooltip contentStyle={{borderRadius: '8px', border: 'none'}} />
                        <Area type="monotone" dataKey="count" stroke="#3b82f6" fillOpacity={1} fill="url(#colorCount)" />
                    </AreaChart>
                </ResponsiveContainer>
            ) : (
                <div className="h-full flex items-center justify-center text-slate-400">No time-series data available</div>
            )}
        </div>
    </Card>

    <UserTableModal isOpen={showUsersModal} onClose={() => setShowUsersModal(false)} users={data?.conversations || []} />
  </div>
  );
};

// ... (Rest of existing components like MessageInspector, ConversationsTab, PerformanceTab, SettingsTab, App)
const MessageInspector = ({ message }) => {
    if (!message) {
        return (
            <div className="h-full flex flex-col items-center justify-center text-slate-400 dark:text-slate-500 p-8 text-center bg-slate-50/50 dark:bg-slate-900/50">
                <Search className="h-10 w-10 mb-4 opacity-20" />
                <p className="text-sm font-medium">Select a message to inspect</p>
                <p className="text-xs mt-1">View entities, slots, and flow triggers</p>
            </div>
        );
    }

    return (
        <div className="h-full flex flex-col bg-white dark:bg-slate-900 border-l border-slate-200 dark:border-slate-800 animate-in slide-in-from-right-10 duration-200 w-[300px]">
            <div className="p-4 border-b border-slate-100 dark:border-slate-800 flex justify-between items-center bg-slate-50 dark:bg-slate-800/50">
                <span className="text-xs font-bold uppercase tracking-wider text-slate-500 dark:text-slate-400 flex items-center gap-2">
                    <Eye className="h-3 w-3" /> Inspection
                </span>
                <span className="text-xs text-slate-400 font-mono">{new Date(message.timestamp * 1000).toLocaleTimeString()}</span>
            </div>
            
            <div className="flex-1 overflow-y-auto p-4 space-y-6 dark:text-slate-300">
                <div>
                    <h4 className="text-xs font-semibold text-slate-900 dark:text-slate-100 mb-2 flex items-center gap-1"><MessageSquare className="h-3 w-3" /> Content</h4>
                    <div className="bg-slate-50 dark:bg-slate-800 p-3 rounded-lg text-sm text-slate-700 dark:text-slate-300 border border-slate-100 dark:border-slate-700">
                        {message.text}
                    </div>
                </div>

                {message.role === 'user' && (
                    <div>
                        <h4 className="text-xs font-semibold text-slate-900 dark:text-slate-100 mb-2 flex items-center gap-1"><Zap className="h-3 w-3" /> Intent</h4>
                        {message.intent ? (
                            <div className="flex items-center justify-between bg-emerald-50 dark:bg-emerald-900/20 p-2 rounded border border-emerald-100 dark:border-emerald-900/30">
                                <span className="text-sm font-medium text-emerald-800 dark:text-emerald-400">{message.intent}</span>
                                <span className="text-xs font-bold text-emerald-600 dark:text-emerald-500">{((message.conf || 0) * 100).toFixed(0)}%</span>
                            </div>
                        ) : (
                            <div className="text-xs text-slate-400 italic">No intent detected</div>
                        )}
                    </div>
                )}

                {message.triggered_flow && (
                    <div>
                        <h4 className="text-xs font-semibold text-slate-900 dark:text-slate-100 mb-2 flex items-center gap-1"><GitBranch className="h-3 w-3" /> Triggered Flow</h4>
                        <div className="flex items-center gap-2 bg-indigo-50 dark:bg-indigo-900/20 p-2 rounded border border-indigo-100 dark:border-indigo-900/30">
                            <GitBranch className="h-4 w-4 text-indigo-600 dark:text-indigo-400" />
                            <span className="text-sm font-medium text-indigo-800 dark:text-indigo-300">{message.triggered_flow}</span>
                        </div>
                    </div>
                )}

                {message.entities && message.entities.length > 0 && (
                    <div>
                        <h4 className="text-xs font-semibold text-slate-900 dark:text-slate-100 mb-2 flex items-center gap-1"><Tag className="h-3 w-3" /> Entities</h4>
                        <div className="space-y-2">
                            {message.entities.map((ent, i) => (
                                <div key={i} className="flex flex-col bg-white dark:bg-slate-800 border border-slate-200 dark:border-slate-700 rounded p-2 text-xs shadow-sm">
                                    <div className="flex justify-between mb-1">
                                        <span className="font-semibold text-blue-600 dark:text-blue-400">{ent.entity}</span>
                                        <span className="text-slate-400">{((ent.confidence||0)*100).toFixed(0)}%</span>
                                    </div>
                                    <div className="text-slate-700 dark:text-slate-300 bg-slate-50 dark:bg-slate-700/50 px-1 rounded">{ent.value}</div>
                                </div>
                            ))}
                        </div>
                    </div>
                )}

                {message.slots_set && message.slots_set.length > 0 && (
                    <div>
                        <h4 className="text-xs font-semibold text-slate-900 dark:text-slate-100 mb-2 flex items-center gap-1"><Database className="h-3 w-3" /> Slots Set</h4>
                        <div className="space-y-1">
                             {message.slots_set.map((slot, i) => (
                                 <div key={i} className="flex items-center justify-between text-xs p-1.5 border-b border-slate-100 dark:border-slate-800 last:border-0">
                                     <span className="text-slate-500 dark:text-slate-400">{slot.name}</span>
                                     <span className="font-mono font-medium text-purple-700 dark:text-purple-400">{String(slot.value)}</span>
                                 </div>
                             ))}
                        </div>
                    </div>
                )}
            </div>
        </div>
    );
};

const ConversationsTab = ({ conversations, settings, filterType, onClearFilter, onFlagSession, onSimulateMessage }) => {
  const [selectedId, setSelectedId] = useState(null);
  const [searchTerm, setSearchTerm] = useState("");
  const [simInput, setSimInput] = useState("");
  const [isBotTyping, setIsBotTyping] = useState(false);
  const [inspectedMessage, setInspectedMessage] = useState(null);
  const [viewMode, setViewMode] = useState("transcript"); // 'transcript' | 'raw'
  const scrollRef = useRef(null);
  const [openRawDetails, setOpenRawDetails] = useState({}); // Track expanded raw details

  const isLowConfidence = (conf) => conf < (settings.confidenceThreshold || 0.6);

  const filteredConversations = useMemo(() => {
    let baseList = conversations || [];
    if (filterType === 'review') {
      baseList = baseList.filter(c => c.latest_intent === 'nlu_fallback' || isLowConfidence(c.confidence));
    } else if (filterType === 'flagged') {
      baseList = baseList.filter(c => c.flagged);
    }
    let searched = baseList.filter(c => 
      c.id.toLowerCase().includes(searchTerm.toLowerCase()) || 
      c.latest_intent.toLowerCase().includes(searchTerm.toLowerCase())
    );
    return searched;
  }, [conversations, searchTerm, filterType, settings.confidenceThreshold]);

  useEffect(() => {
    if (filteredConversations && filteredConversations.length > 0) {
        if (!filteredConversations.find(c => c.id === selectedId)) {
             setSelectedId(filteredConversations[0].id);
             setInspectedMessage(null); 
             setOpenRawDetails({});
        }
    } else {
        setSelectedId(null);
    }
  }, [filteredConversations]);

  const activeConversation = conversations?.find(c => c.id === selectedId);

  useEffect(() => {
    if (scrollRef.current) scrollRef.current.scrollTop = scrollRef.current.scrollHeight;
  }, [activeConversation?.history, isBotTyping, viewMode]);

  const handleSendSimulation = () => {
    if (!simInput.trim()) return;
    onSimulateMessage(selectedId, simInput);
    setSimInput("");
    setIsBotTyping(true);
    setTimeout(() => setIsBotTyping(false), 1500);
  };

  const toggleRawDetail = (idx) => {
      setOpenRawDetails(prev => ({...prev, [idx]: !prev[idx]}));
  };

  const getEventIcon = (evtName) => {
      if (evtName === 'user') return <User className="h-4 w-4" />;
      if (evtName === 'bot') return <Bot className="h-4 w-4" />;
      if (evtName === 'action') return <Zap className="h-4 w-4" />;
      if (evtName === 'slot') return <Tag className="h-4 w-4" />;
      if (evtName.includes('flow')) return <GitBranch className="h-4 w-4" />;
      return <Code className="h-4 w-4" />;
  };

  const getEventColor = (evtName) => {
      if (evtName === 'user') return "bg-white dark:bg-slate-800 border-l-4 border-l-slate-900 dark:border-l-slate-400";
      if (evtName === 'bot') return "bg-white dark:bg-slate-800 border-l-4 border-l-blue-500";
      if (evtName === 'action') return "bg-amber-50 dark:bg-amber-900/10 border-l-4 border-l-amber-500";
      if (evtName === 'slot') return "bg-purple-50 dark:bg-purple-900/10 border-l-4 border-l-purple-500";
      if (evtName.includes('flow')) return "bg-indigo-50 dark:bg-indigo-900/10 border-l-4 border-l-indigo-500";
      return "bg-slate-50 dark:bg-slate-900 border-l-4 border-l-slate-300";
  };

  return (
    <div className="flex flex-col h-[calc(100vh-140px)] animate-in fade-in duration-300">
      
      {filterType && filterType !== 'all' && (
        <div className="bg-slate-900 text-white px-4 py-2 flex items-center justify-between text-sm rounded-t-xl mx-1">
          <div className="flex items-center gap-2">
            {filterType === 'flagged' ? <Flag className="h-4 w-4" /> : <Filter className="h-4 w-4" />}
            <span className="font-medium capitalize">
              Filtered View: {filterType}
            </span>
          </div>
          <button onClick={onClearFilter} className="text-slate-300 hover:text-white underline text-xs">Clear Filter</button>
        </div>
      )}

      <div className={`flex flex-1 border border-slate-200 dark:border-slate-800 ${filterType && filterType !== 'all' ? 'rounded-b-xl border-t-0' : 'rounded-xl'} overflow-hidden bg-white dark:bg-slate-900 shadow-sm`}>
        {/* LEFT: Conversation List */}
        <div className="w-1/4 border-r border-slate-200 dark:border-slate-800 flex flex-col min-w-[250px]">
          <div className="p-4 border-b border-slate-100 dark:border-slate-800 bg-slate-50/50 dark:bg-slate-800/20">
            <div className="relative">
              <Search className="absolute left-2.5 top-2.5 h-4 w-4 text-slate-400" />
              <Input 
                placeholder="Search..." 
                className="pl-9 bg-white dark:bg-slate-900" 
                value={searchTerm}
                onChange={(e) => setSearchTerm(e.target.value)}
              />
            </div>
          </div>
          <div className="flex-1 overflow-y-auto">
            {filteredConversations?.length === 0 && (
              <div className="p-8 text-center text-slate-400 text-sm">No conversations found.</div>
            )}
            {filteredConversations?.map((conv) => (
              <div 
                key={conv.id}
                onClick={() => { setSelectedId(conv.id); setInspectedMessage(null); }}
                className={`p-4 border-b border-slate-50 dark:border-slate-800 cursor-pointer transition-colors hover:bg-slate-50 dark:hover:bg-slate-800/50 ${selectedId === conv.id ? 'bg-slate-50 dark:bg-slate-800/80 ring-1 ring-inset ring-slate-200 dark:ring-slate-700' : ''}`}
              >
                <div className="flex justify-between items-start mb-1">
                  <div className="flex items-center gap-2 overflow-hidden">
                      {conv.flagged && <Flag className="h-3 w-3 text-red-500 fill-red-500 shrink-0" />}
                      <span className="font-medium text-sm text-slate-900 dark:text-slate-200 truncate">{conv.id}</span>
                  </div>
                  <span className="text-xs text-slate-400 shrink-0">{conv.timestamp.split(',')[0]}</span>
                </div>
                <div className="flex flex-wrap items-center gap-2 mb-2">
                  <Badge variant={conv.latest_intent === 'nlu_fallback' || isLowConfidence(conv.confidence) ? 'destructive' : 'secondary'}>
                    {conv.latest_intent}
                  </Badge>
                  {conv.active_flow && (
                    <Badge variant="indigo" className="text-[10px] px-1.5 py-0 flex items-center gap-1">
                        <GitBranch className="h-3 w-3" /> {conv.active_flow}
                    </Badge>
                  )}
                  {conv.channel !== 'unknown' && (
                     <Badge variant="outline" className="text-[10px] px-1.5 py-0">
                        {conv.channel}
                     </Badge>
                  )}
                </div>
              </div>
            ))}
          </div>
        </div>

        {/* MIDDLE: Chat Transcript */}
        {activeConversation ? (
          <div className="flex-1 flex flex-col bg-slate-50/30 dark:bg-slate-950/50 min-w-0 relative">
            <div className="p-4 border-b border-slate-200 dark:border-slate-800 bg-white dark:bg-slate-900 flex justify-between items-center shrink-0">
              <div>
                <h2 className="font-semibold text-slate-900 dark:text-slate-100 flex items-center gap-2">
                  <User className="h-4 w-4 text-slate-400" />
                  {activeConversation.id}
                </h2>
                <div className="flex items-center gap-2 mt-1">
                  <span className="flex items-center gap-1 text-xs text-slate-500 dark:text-slate-400">
                    <Activity className="h-3 w-3" /> Status: 
                    <span className={isLowConfidence(activeConversation.confidence) || activeConversation.flagged ? "text-red-500 font-medium ml-1" : "text-emerald-600 font-medium ml-1"}>
                        {activeConversation.flagged ? 'Flagged' : isLowConfidence(activeConversation.confidence) ? 'Needs Review' : 'Active'}
                    </span>
                  </span>
                </div>
              </div>
              <div className="flex items-center gap-2">
                 <div className="flex bg-slate-100 dark:bg-slate-800 rounded-md p-0.5 border border-slate-200 dark:border-slate-700 mr-2">
                    <button 
                        onClick={() => setViewMode('transcript')}
                        className={`px-2 py-1 text-xs font-medium rounded-sm transition-colors ${viewMode === 'transcript' ? 'bg-white dark:bg-slate-600 text-slate-900 dark:text-slate-100 shadow-sm' : 'text-slate-500 dark:text-slate-400'}`}
                    >Transcript</button>
                    <button 
                        onClick={() => setViewMode('raw')}
                        className={`px-2 py-1 text-xs font-medium rounded-sm transition-colors ${viewMode === 'raw' ? 'bg-white dark:bg-slate-600 text-slate-900 dark:text-slate-100 shadow-sm' : 'text-slate-500 dark:text-slate-400'}`}
                    >Raw Events</button>
                 </div>
                 <Button 
                    variant={activeConversation.flagged ? "danger" : "dangerOutline"} 
                    className="h-8 justify-center gap-2 transition-all"
                    onClick={() => onFlagSession(activeConversation.id)}
                 >
                    <Flag className={`h-3 w-3 ${activeConversation.flagged ? 'fill-white' : ''}`} />
                 </Button>
              </div>
            </div>

            <div className="flex-1 overflow-y-auto p-6 space-y-6" ref={scrollRef}>
                {viewMode === 'raw' ? (
                     <div className="space-y-4 relative before:absolute before:left-4 before:top-2 before:bottom-2 before:w-px before:bg-slate-200 dark:before:bg-slate-800">
                      {activeConversation.raw_events?.map((evt, idx) => (
                          <div key={idx} className="relative pl-10">
                              <div className="absolute left-2 top-3 w-4 h-4 rounded-full bg-white dark:bg-slate-900 border-2 border-slate-300 dark:border-slate-700 flex items-center justify-center z-10">
                                  <div className={`w-1.5 h-1.5 rounded-full ${evt.event === 'user' ? 'bg-slate-800' : 'bg-slate-400'}`}></div>
                              </div>
                              <div className={`border rounded-lg shadow-sm text-sm overflow-hidden ${getEventColor(evt.event)} border-slate-200 dark:border-slate-800`}>
                                  <div className="p-3">
                                      <div className="flex items-center justify-between mb-2">
                                          <div className="flex items-center gap-2 font-medium text-slate-900 dark:text-slate-200 capitalize">
                                              {getEventIcon(evt.event)}
                                              {evt.event.replace(/_/g, ' ')}
                                          </div>
                                          <div className="flex items-center gap-2">
                                              <span className="text-xs text-slate-400 font-mono">
                                                  {new Date(evt.timestamp * 1000).toLocaleTimeString()}
                                              </span>
                                              <button 
                                                onClick={() => toggleRawDetail(idx)}
                                                className="text-slate-400 hover:text-slate-600 dark:hover:text-slate-200"
                                              >
                                                  <List className="h-3 w-3" />
                                              </button>
                                          </div>
                                      </div>
                                      
                                      {/* CONTENT RENDERER */}
                                      <div className="text-slate-700 dark:text-slate-300">
                                          {evt.event === 'user' && (
                                              <div>
                                                  <div className="font-medium">"{evt.text}"</div>
                                                  <div className="mt-2 flex flex-wrap gap-2">
                                                      {evt.parse_data?.intent && (
                                                          <span className="inline-flex items-center px-1.5 py-0.5 rounded text-xs font-medium bg-white dark:bg-slate-800 border border-slate-200 dark:border-slate-700 text-slate-600 dark:text-slate-400">
                                                              <Zap className="h-3 w-3 mr-1" />
                                                              {evt.parse_data.intent.name} 
                                                              <span className="ml-1 text-slate-400">({(evt.parse_data.intent.confidence * 100).toFixed(0)}%)</span>
                                                          </span>
                                                      )}
                                                      {evt.parse_data?.entities?.map((e, i) => (
                                                          <span key={i} className="inline-flex items-center px-1.5 py-0.5 rounded text-xs font-medium bg-blue-50 dark:bg-blue-900/20 text-blue-700 dark:text-blue-300 border border-blue-100 dark:border-blue-900/30">
                                                              <Tag className="h-3 w-3 mr-1" />
                                                              {e.entity}: {e.value}
                                                          </span>
                                                      ))}
                                                  </div>
                                              </div>
                                          )}
                                          {evt.event === 'bot' && (
                                              <div className="italic">"{evt.text}"</div>
                                          )}
                                          {evt.event === 'action' && (
                                              <div className="font-mono text-amber-700 dark:text-amber-400">{evt.name}</div>
                                          )}
                                          {evt.event === 'slot' && (
                                              <div className="font-mono text-purple-700 dark:text-purple-400">
                                                  {evt.name} <span className="text-slate-400">←</span> {JSON.stringify(evt.value)}
                                              </div>
                                          )}
                                          {evt.event === 'flow_started' && (
                                              <div className="font-mono text-indigo-700 dark:text-indigo-400 font-bold">
                                                  {evt.flow_id}
                                              </div>
                                          )}
                                      </div>
                                  </div>

                                  {/* EXPANDABLE RAW JSON */}
                                  {openRawDetails[idx] && (
                                      <div className="border-t border-slate-200 dark:border-slate-800 bg-slate-50 dark:bg-slate-950 p-3">
                                          <pre className="text-[10px] text-slate-500 dark:text-slate-400 font-mono overflow-x-auto whitespace-pre-wrap">
                                              {JSON.stringify(evt, null, 2)}
                                          </pre>
                                      </div>
                                  )}
                              </div>
                          </div>
                      ))}
                  </div>
                ) : (
                    <>
                    {activeConversation.history?.map((msg, idx) => (
                    <div key={idx} className={`flex gap-4 group ${msg.role === 'user' ? 'justify-end' : 'justify-start'}`}>
                        {msg.role === 'bot' && (
                            <div className="w-8 h-8 rounded-full bg-slate-200 dark:bg-slate-800 flex items-center justify-center shrink-0 mt-1">
                                <Bot className="h-4 w-4 text-slate-600 dark:text-slate-300" />
                            </div>
                        )}
                        <div 
                            className={`max-w-[70%] space-y-1 ${msg.role === 'user' ? 'items-end flex flex-col' : ''}`}
                            onClick={() => setInspectedMessage(msg)}
                        >
                            {msg.triggered_flow && (
                                <div className="flex items-center gap-1 text-[10px] text-indigo-600 dark:text-indigo-400 font-medium mb-1 bg-indigo-50 dark:bg-indigo-900/30 px-2 py-0.5 rounded-full border border-indigo-100 dark:border-indigo-900/50 self-start">
                                    <GitBranch className="h-3 w-3" />
                                    Starts Flow: {msg.triggered_flow}
                                </div>
                            )}
                            
                            <div className={`p-3 rounded-2xl text-sm shadow-sm cursor-pointer transition-all border-2 ${
                                msg === inspectedMessage ? 'ring-2 ring-blue-400 ring-offset-1' : 'border-transparent'
                            } ${
                                msg.role === 'user' 
                                    ? 'bg-slate-900 dark:bg-slate-700 text-white rounded-br-none hover:bg-slate-800 dark:hover:bg-slate-600' 
                                    : 'bg-white dark:bg-slate-800 border-slate-200 dark:border-slate-700 text-slate-700 dark:text-slate-200 rounded-bl-none hover:border-slate-300'
                            }`}>
                                {msg.text}
                            </div>
                            
                            <div className="flex items-center gap-2 px-1">
                                {msg.entities?.length > 0 && (
                                    <span className="text-[10px] text-blue-500 font-medium flex items-center gap-0.5">
                                        <Tag className="h-3 w-3" /> {msg.entities.length}
                                    </span>
                                )}
                                {msg.slots_set?.length > 0 && (
                                    <span className="text-[10px] text-purple-500 font-medium flex items-center gap-0.5">
                                        <Database className="h-3 w-3" /> {msg.slots_set.length}
                                    </span>
                                )}
                                <span className="text-[10px] text-slate-300 dark:text-slate-600 group-hover:text-slate-400 transition-colors">
                                    {new Date(msg.timestamp * 1000).toLocaleTimeString([], {hour: '2-digit', minute:'2-digit'})}
                                </span>
                            </div>
                        </div>
                        {msg.role === 'user' && (
                            <div className="w-8 h-8 rounded-full bg-slate-100 dark:bg-slate-800 flex items-center justify-center shrink-0 mt-1">
                                <User className="h-4 w-4 text-slate-400 dark:text-slate-500" />
                            </div>
                        )}
                    </div>
                ))}
                </>
                )}
            </div>
            
            <div className="p-4 bg-white dark:bg-slate-900 border-t border-slate-200 dark:border-slate-800 shrink-0">
              <div className="flex gap-2 items-center">
                <Input 
                    placeholder="Type to simulate response..." 
                    className="flex-1" 
                    value={simInput}
                    onChange={(e) => setSimInput(e.target.value)}
                    onKeyDown={(e) => e.key === 'Enter' && handleSendSimulation()}
                />
                <Button 
                    variant="primary" 
                    className="h-9 w-9 p-0 rounded-full shrink-0"
                    onClick={handleSendSimulation}
                >
                    <Send className="h-4 w-4" />
                </Button>
              </div>
            </div>
          </div>
        ) : (
          <div className="flex-1 flex items-center justify-center text-slate-400 dark:text-slate-500 bg-slate-50/30 dark:bg-slate-950/30">
            Select a conversation to inspect
          </div>
        )}

        {/* RIGHT: Inspector Panel */}
        {activeConversation && <MessageInspector message={inspectedMessage} />}
      </div>
    </div>
  );
};

const PerformanceTab = ({ data }) => {
    if (!data) return null;
    
    return (
    <div className="space-y-8 animate-in fade-in duration-500">
      
      <div className="flex justify-between items-end">
          <div>
            <h2 className="text-lg font-bold text-slate-900 dark:text-slate-100 flex items-center gap-2">
                <Zap className="h-5 w-5 text-indigo-600 dark:text-indigo-400" />
                Event & Performance Analytics
            </h2>
            <p className="text-sm text-slate-500 dark:text-slate-400 mt-1">Deep dive into system latency, slot usage, and flow completion.</p>
          </div>
      </div>
  
      <div className="grid grid-cols-1 md:grid-cols-4 gap-6">
         <Card className="p-6">
           <div className="text-sm font-medium text-slate-500 dark:text-slate-400 mb-2">Total Events</div>
           <div className="text-3xl font-bold text-slate-900 dark:text-slate-100">{data?.metrics?.totalEvents || 0}</div>
         </Card>
         <Card className="p-6">
           <div className="text-sm font-medium text-slate-500 dark:text-slate-400 mb-2">Avg Latency</div>
           <div className="text-3xl font-bold text-slate-900 dark:text-slate-100">{Math.round(data?.metrics?.avgLatency || 0)}ms</div>
           <div className="text-xs text-emerald-600 dark:text-emerald-400 mt-1">Acceptable range</div>
         </Card>
         <Card className="p-6">
           <div className="text-sm font-medium text-slate-500 dark:text-slate-400 mb-2">Flow Success</div>
           <div className="text-3xl font-bold text-slate-900 dark:text-slate-100">{((data?.metrics?.flowCompletionRate || 0) * 100).toFixed(1)}%</div>
         </Card>
         <Card className="p-6">
           <div className="text-sm font-medium text-slate-500 dark:text-slate-400 mb-2">Top Action</div>
           <div className="text-xl font-bold text-indigo-600 dark:text-indigo-400 truncate">{data?.charts?.actions?.[0]?.name || "N/A"}</div>
           <div className="text-xs text-slate-400 mt-1">Most triggered</div>
         </Card>
      </div>
  
      <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
        <Card className="p-6">
          <div className="mb-4">
            <h3 className="font-semibold text-slate-900 dark:text-slate-100">Bot Response Latency</h3>
            <p className="text-sm text-slate-500 dark:text-slate-400">Average processing time (ms) per hour.</p>
          </div>
          <div className="h-[300px] w-full">
            <ResponsiveContainer width="100%" height="100%">
              <AreaChart data={data?.charts?.latency || []}>
                 <CartesianGrid strokeDasharray="3 3" vertical={false} stroke="#f1f5f9" className="dark:opacity-10" />
                 <XAxis dataKey="time" axisLine={false} tickLine={false} tick={{fill: '#64748b', fontSize: 12}} />
                 <YAxis axisLine={false} tickLine={false} tick={{fill: '#64748b', fontSize: 12}} />
                 <Tooltip contentStyle={{ borderRadius: '8px', border: 'none', backgroundColor: '#fff', color: '#000' }} />
                 <Area type="monotone" dataKey="ms" stroke="#6366f1" fill="#e0e7ff" />
              </AreaChart>
            </ResponsiveContainer>
          </div>
        </Card>
  
        <Card className="p-6">
          <div className="mb-4">
            <h3 className="font-semibold text-slate-900 dark:text-slate-100">Action Distribution</h3>
            <p className="text-sm text-slate-500 dark:text-slate-400">Frequency of bot actions & custom code.</p>
          </div>
          <div className="h-[300px] w-full">
            <ResponsiveContainer width="100%" height="100%">
               <BarChart data={data?.charts?.actions || []} layout="vertical">
                  <CartesianGrid strokeDasharray="3 3" horizontal={true} vertical={false} stroke="#f1f5f9" className="dark:opacity-10" />
                  <XAxis type="number" hide />
                  <YAxis dataKey="name" type="category" width={140} tick={{fontSize: 11, fill: '#64748b'}} />
                  <Tooltip cursor={{fill: 'transparent'}} contentStyle={{ borderRadius: '8px', border: 'none' }} />
                  <Bar dataKey="value" fill="#f59e0b" radius={[0, 4, 4, 0]} barSize={20} />
               </BarChart>
            </ResponsiveContainer>
          </div>
        </Card>
      </div>

       {/* NEW: Events Per Session Chart */}
      <Card className="p-6">
          <div className="mb-4 flex justify-between items-center">
            <div>
                <h3 className="font-semibold text-slate-900 dark:text-slate-100">Events per Conversation (Top 50)</h3>
                <p className="text-sm text-slate-500 dark:text-slate-400">Conversation complexity by event count. Identifies heavy sessions.</p>
            </div>
            <BarChart2 className="h-5 w-5 text-slate-400" />
          </div>
          <div className="h-[300px] w-full">
            <ResponsiveContainer width="100%" height="100%">
               <BarChart data={data?.charts?.sessionComplexity || []}>
                  <CartesianGrid strokeDasharray="3 3" vertical={false} stroke="#f1f5f9" className="dark:opacity-10" />
                  <XAxis dataKey="id" axisLine={false} tick={false} label={{ value: 'Conversation IDs', position: 'insideBottom', offset: -5, fill: '#94a3b8' }} />
                  <YAxis axisLine={false} tickLine={false} tick={{fill: '#64748b', fontSize: 12}} />
                  <Tooltip 
                      cursor={{fill: '#f1f5f9'}} 
                      contentStyle={{ borderRadius: '8px', border: 'none' }} 
                      labelStyle={{ color: '#64748b', marginBottom: '5px' }}
                  />
                  <Bar dataKey="eventCount" name="Total Events" fill="#8b5cf6" radius={[4, 4, 0, 0]} />
                  <Bar dataKey="turnCount" name="User Turns" fill="#c4b5fd" radius={[4, 4, 0, 0]} />
                  <Legend />
               </BarChart>
            </ResponsiveContainer>
          </div>
      </Card>
    </div>
    );
};

const SettingsTab = ({ settings, updateSettings, onClearData }) => {
    return (
      <div className="max-w-2xl mx-auto space-y-8 animate-in fade-in slide-in-from-bottom-2 duration-500">
        <Card className="p-6">
            <div className="flex items-center gap-3 mb-6 border-b border-slate-100 dark:border-slate-800 pb-4">
            <div className="p-2 bg-slate-100 dark:bg-slate-800 rounded-lg"><Sliders className="h-5 w-5 text-slate-700 dark:text-slate-300" /></div>
            <div>
                <h3 className="font-semibold text-slate-900 dark:text-slate-100">Analysis Thresholds</h3>
                <p className="text-sm text-slate-500 dark:text-slate-400">Configure how RasaLens interprets data.</p>
            </div>
            </div>
            
            <div className="space-y-6">
            <div className="space-y-2">
                <div className="flex justify-between">
                <label className="text-sm font-medium text-slate-700 dark:text-slate-300">Low Confidence Threshold</label>
                <span className="text-sm font-bold text-slate-900 dark:text-slate-100">{settings.confidenceThreshold}</span>
                </div>
                <input 
                type="range" 
                min="0.1" 
                max="1.0" 
                step="0.05"
                value={settings.confidenceThreshold}
                onChange={(e) => updateSettings({ ...settings, confidenceThreshold: parseFloat(e.target.value) })}
                className="w-full h-2 bg-slate-200 dark:bg-slate-700 rounded-lg appearance-none cursor-pointer accent-slate-900 dark:accent-slate-500"
                />
            </div>
            </div>
        </Card>

        <Card className="p-6">
            <div className="flex items-center gap-3 mb-6 border-b border-slate-100 dark:border-slate-800 pb-4">
            <div className="p-2 bg-slate-100 dark:bg-slate-800 rounded-lg"><Clock className="h-5 w-5 text-slate-700 dark:text-slate-300" /></div>
            <div>
                <h3 className="font-semibold text-slate-900 dark:text-slate-100">Data Retention</h3>
                <p className="text-sm text-slate-500 dark:text-slate-400">Manage locally cached analysis.</p>
            </div>
            </div>
            
            <div className="flex items-center justify-between">
            <div className="space-y-1">
                <div className="font-medium text-sm text-slate-900 dark:text-slate-100">Clear Cache</div>
                <div className="text-xs text-slate-500 dark:text-slate-400">Remove all data from local storage.</div>
            </div>
            <Button onClick={onClearData} variant="outline" className="text-red-600 hover:text-red-700 hover:bg-red-50 dark:text-red-400 dark:border-red-900 dark:hover:bg-red-900/20 justify-center">Clear Data</Button>
            </div>
        </Card>
      </div>
    );
};

// --- MAIN APP ---

export default function App() {
  const [isConnected, setIsConnected] = useState(false);
  const [activeTab, setActiveTab] = useState("overview");
  const [appData, setAppData] = useState(null);
  const [isSidebarCollapsed, setIsSidebarCollapsed] = useState(false);
  const [isDarkMode, setIsDarkMode] = useState(false);
  
  const [conversationFilter, setConversationFilter] = useState('all');
  
  const [settings, setSettings] = useState({
    confidenceThreshold: 0.6,
    sessionTimeout: 60
  });

  useEffect(() => {
      const storedData = db.load();
      if (storedData) {
          setAppData(storedData);
          setIsConnected(true);
      }
      // Removed automatic dark mode detection to enforce light mode default
  }, []);

  const handleUpdateSettings = (newSettings) => {
      setSettings(newSettings);
      db.saveSettings(newSettings);
  };

  const handleClearData = () => {
      db.clear();
      setAppData(null);
      setIsConnected(false);
  };

  const handleFlagSession = (id) => {
      if (!appData) return;
      const updated = appData.conversations.map(c => c.id === id ? {...c, flagged: !c.flagged} : c);
      const newData = { ...appData, conversations: updated };
      setAppData(newData);
      db.save(newData);
  };

  const handleSimulateMessage = (id, text) => {
      if (!appData) return;
      const newMsg = {
          role: 'user',
          text: text,
          timestamp: Date.now() / 1000,
          simulated: true
      };
      
      let updatedConversations = appData.conversations.map(c => {
          if (c.id === id) return { ...c, history: [...c.history, newMsg] };
          return c;
      });
      const newData = { ...appData, conversations: updatedConversations };
      setAppData(newData);
      db.save(newData);
      
      setTimeout(() => {
           const botMsg = {
              role: 'bot',
              text: `[Simulated] I received: "${text}". How else can I help?`,
              timestamp: Date.now() / 1000,
              simulated: true
          };
          const updatedWithBot = newData.conversations.map(c => {
            if (c.id === id) return { ...c, history: [...c.history, botMsg] };
            return c;
          });
          const finalData = { ...newData, conversations: updatedWithBot };
          setAppData(finalData);
          db.save(finalData);
      }, 1500);
  };
  
  const handleDrillDown = (type) => {
      if (type === 'performance') setActiveTab('performance');
      else { setActiveTab('conversations'); setConversationFilter('all'); }
  };

  if (!isConnected) {
    return (
        <div className={isDarkMode ? "dark" : ""}>
             <ConnectionGateway onConnect={(type, rawData) => {
                const processed = processData(rawData || []); 
                if (processed) { setAppData(processed); db.save(processed); }
                setIsConnected(true);
            }} />
        </div>
    );
  }

  return (
    <div className={isDarkMode ? "dark" : ""}>
      <div className="flex h-screen bg-slate-50 dark:bg-slate-950 text-slate-900 dark:text-slate-100 font-sans overflow-hidden transition-colors duration-200">
        
        {/* Sidebar */}
        <aside className={`${isSidebarCollapsed ? 'w-16' : 'w-64'} bg-white dark:bg-slate-900 border-r border-slate-200 dark:border-slate-800 flex flex-col transition-all duration-300 ease-in-out shrink-0`}>
          <div className={`p-6 border-b border-slate-100 dark:border-slate-800 flex items-center ${isSidebarCollapsed ? 'justify-center' : 'justify-between'}`}>
            {!isSidebarCollapsed && (
              <div className="flex items-center gap-2 font-bold text-xl tracking-tight text-slate-900 dark:text-slate-100 whitespace-nowrap overflow-hidden">
                <div className="w-8 h-8 bg-slate-900 dark:bg-slate-100 rounded-lg flex items-center justify-center text-white dark:text-slate-900 shadow-lg shadow-slate-200 dark:shadow-none shrink-0">R</div>
                RasaLens
              </div>
            )}
            {isSidebarCollapsed && (
              <div className="w-8 h-8 bg-slate-900 dark:bg-slate-100 rounded-lg flex items-center justify-center text-white dark:text-slate-900 shadow-lg shadow-slate-200 dark:shadow-none shrink-0">R</div>
            )}
            <button onClick={() => setIsSidebarCollapsed(!isSidebarCollapsed)} className={`p-1 rounded-md hover:bg-slate-100 dark:hover:bg-slate-800 text-slate-400 ${isSidebarCollapsed ? 'hidden' : 'block'}`}>
              <ChevronLeft className="h-4 w-4" />
            </button>
          </div>
          
          <nav className="flex-1 p-4 space-y-1 overflow-hidden">
            <Button variant={activeTab === "overview" ? "secondary" : "ghost"} className={`w-full justify-start ${isSidebarCollapsed ? 'px-2' : ''}`} onClick={() => setActiveTab("overview")}>
              <LayoutDashboard className={`h-4 w-4 ${isSidebarCollapsed ? 'mr-0' : 'mr-2'}`} /> {!isSidebarCollapsed && "Overview"}
            </Button>
            <Button variant={activeTab === "conversations" ? "secondary" : "ghost"} className={`w-full justify-start ${isSidebarCollapsed ? 'px-2' : ''}`} onClick={() => { setActiveTab("conversations"); setConversationFilter('all'); }}>
              <MessageSquare className={`h-4 w-4 ${isSidebarCollapsed ? 'mr-0' : 'mr-2'}`} /> {!isSidebarCollapsed && "Conversations"}
            </Button>
            <Button variant={activeTab === "flagged" ? "secondary" : "ghost"} className={`w-full justify-start ${isSidebarCollapsed ? 'px-2' : ''}`} onClick={() => { setActiveTab("conversations"); setConversationFilter('flagged'); }}>
              <Flag className={`h-4 w-4 ${isSidebarCollapsed ? 'mr-0' : 'mr-2'}`} /> {!isSidebarCollapsed && "Flagged"}
            </Button>
            <Button variant={activeTab === "performance" ? "secondary" : "ghost"} className={`w-full justify-start ${isSidebarCollapsed ? 'px-2' : ''}`} onClick={() => setActiveTab("performance")}>
              <Activity className={`h-4 w-4 ${isSidebarCollapsed ? 'mr-0' : 'mr-2'}`} /> {!isSidebarCollapsed && "Performance"}
            </Button>
            
            <div className="pt-4 mt-auto border-t border-slate-100 dark:border-slate-800 space-y-1">
                <Button variant={activeTab === "settings" ? "secondary" : "ghost"} className={`w-full justify-start ${isSidebarCollapsed ? 'px-2' : ''}`} onClick={() => setActiveTab("settings")}>
                <Settings className={`h-4 w-4 ${isSidebarCollapsed ? 'mr-0' : 'mr-2'}`} /> {!isSidebarCollapsed && "Settings"}
                </Button>
                <Button variant="ghost" className={`w-full justify-start ${isSidebarCollapsed ? 'px-2' : ''}`} onClick={() => setIsDarkMode(!isDarkMode)}>
                {isDarkMode ? <Sun className={`h-4 w-4 ${isSidebarCollapsed ? 'mr-0' : 'mr-2'}`} /> : <Moon className={`h-4 w-4 ${isSidebarCollapsed ? 'mr-0' : 'mr-2'}`} />} 
                {!isSidebarCollapsed && (isDarkMode ? "Light Mode" : "Dark Mode")}
                </Button>
            </div>
          </nav>
        </aside>

        {/* Main Content */}
        <main className="flex-1 flex flex-col h-screen overflow-hidden">
          <header className="h-16 bg-white dark:bg-slate-900 border-b border-slate-200 dark:border-slate-800 flex items-center justify-between px-4 sm:px-8 shrink-0">
            <div className="flex items-center text-slate-500 dark:text-slate-400 text-sm">Dashboard / <span className="text-slate-900 dark:text-slate-100 ml-1 font-medium capitalize">{activeTab}</span></div>
            <Button variant="outline" className="h-8 gap-2 justify-center" onClick={() => setIsConnected(false)}>Change Store</Button>
          </header>

          <div className="flex-1 overflow-y-auto overflow-x-hidden p-4 sm:p-8 bg-slate-50/50 dark:bg-slate-950/50">
            <div className="max-w-7xl mx-auto">
              {activeTab === 'overview' && <OverviewTab data={appData} onDrillDown={handleDrillDown} />}
              {activeTab === 'conversations' && (
                <ConversationsTab 
                  conversations={appData?.conversations} 
                  settings={settings} 
                  filterType={conversationFilter}
                  onClearFilter={() => setConversationFilter('all')}
                  onFlagSession={handleFlagSession}
                  onSimulateMessage={handleSimulateMessage}
                />
              )}
              {activeTab === 'performance' && <PerformanceTab data={appData} />}
              {activeTab === 'settings' && <SettingsTab settings={settings} updateSettings={handleUpdateSettings} onClearData={handleClearData} />}
            </div>
          </div>
        </main>
      </div>
    </div>
  );
}
```