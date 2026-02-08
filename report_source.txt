import React, { useState, useEffect, useRef } from 'react';
import {
Plus, Trash2, ChevronDown, ChevronUp, Save,
AlertTriangle, CheckCircle2,
Building2, Calendar, Download, X, Check,
ShieldCheck, ShieldAlert, Camera, Video,
FileText, Cloud, Upload, Share2, MessageSquare,
ImageIcon, ClipboardList, FolderOpen,
FileJson, FileSpreadsheet, FileCode, File as FilePdf,
ListTodo, AlertCircle, Play, Maximize2, StickyNote,
Eye, EyeOff, FileDown
} from 'lucide-react';
import { initializeApp } from 'firebase/app';
import {
getFirestore, doc, setDoc, onSnapshot, collection,
updateDoc, deleteDoc, addDoc
} from 'firebase/firestore';
import {
getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged
} from 'firebase/auth';

// Firebase & Environment Config
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'fire-alarm-inspector-v3';

const TARGET_FOLDER_URL = "https://drive.google.com/drive/u/0/folders/11lj4d0lemmAo9h1xgfXxLtWviTHD4syF";

const DEVICE_TYPES = [
{ code: 'H', label: 'Manual Pull Station' },
{ code: 'HT', label: 'Heat Detector (Fixed Temp)' },
{ code: 'RHT', label: 'Heat Detector (Rate of Rise)' },
{ code: 'S', label: 'Smoke Detector' },
{ code: 'DS', label: 'Duct Smoke Detector' },
{ code: 'FS', label: 'Flow Switch' },
{ code: 'TS', label: 'Tamper Switch' },
{ code: 'SA', label: 'Smoke Alarm' },
{ code: 'B', label: 'Bell' },
{ code: 'K', label: 'Horn' },
{ code: 'C', label: 'Chime' },
{ code: 'V', label: 'Visual Appliance' },
{ code: 'SP', label: 'Loud Speaker' },
{ code: 'HSP', label: 'Horn Loud Speaker' },
{ code: 'ET', label: 'Emergency Telephone' },
{ code: 'AD', label: 'Ancillary Devices' }
];

const App = () => {
const [user, setUser] = useState(null);
const [inspections, setInspections] = useState([]);
const [allMedia, setAllMedia] = useState([]);
const [activeYear, setActiveYear] = useState('2025');
const [selectedBuildingId, setSelectedBuildingId] = useState(null);
const [view, setView] = useState('years');
const fileInputRef = useRef(null);
const [currentMediaTarget, setCurrentMediaTarget] = useState(null);
const [expandedMedia, setExpandedMedia] = useState(null);

const [dialogue, setDialogue] = useState({
isOpen: false,
title: '',
message: '',
confirmText: 'Yes',
cancelText: 'Cancel',
onConfirm: null,
onCancel: null,
showInput: false,
inputValue: '',
showTypePicker: false,
onTypeSelect: null
});

const [openSections, setOpenSections] = useState({
details: false, prejob: false, control: false, inventory: false, batteries: false, emergencyLights: false, summary: false
});

const toggleSection = (section) => setOpenSections(prev => ({ ...prev, [section]: !prev[section] }));

// Load PDF Libraries
useEffect(() => {
const script1 = document.createElement('script');
script1.src = "https://www.google.com/search?q=https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js";
script1.onload = () => {
const script2 = document.createElement('script');
script2.src = "https://www.google.com/search?q=https://cdnjs.cloudflare.com/ajax/libs/jspdf-autotable/3.5.23/jspdf.plugin.autotable.min.js";
document.head.appendChild(script2);
};
document.head.appendChild(script1);
}, []);

useEffect(() => {
const initAuth = async () => {
if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
await signInWithCustomToken(auth, __initial_auth_token);
} else {
await signInAnonymously(auth);
}
};
initAuth();
const unsubscribe = onAuthStateChanged(auth, setUser);
return () => unsubscribe();
}, []);

useEffect(() => {
if (!user) return;

const qI = collection(db, 'artifacts', appId, 'public', 'data', 'inspections');
const unsubI = onSnapshot(qI, (snapshot) => {
  const docs = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
  if (docs.length === 0) createBaseline();
  else setInspections(docs);
}, (error) => console.error("Firestore Error Inspections:", error));

const qM = collection(db, 'artifacts', appId, 'public', 'data', 'media');
const unsubM = onSnapshot(qM, (snapshot) => {
  const mediaDocs = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
  setAllMedia(mediaDocs);
}, (error) => console.error("Firestore Error Media:", error));

return () => { unsubI(); unsubM(); };


}, [user]);

const createBaseline = async () => {
const baselineId = 'church-building-001';
const baseline = {
year: '2025',
name: 'LDS Church Welling FM Group',
inspector: '', // Blank to start as requested
address: 'Champion, Alberta',
isMonitored: true,
monitoringPhone: '403-555-0199',
panelManufacturer: 'Edwards',
panelModel: 'Fireshield Plus',
// PRE-JOB: 5 ITEMS
preJob: [
{ id: 'pj1', label: 'Auxiliary functions handled?', value: true, notes: '' },
{ id: 'pj2', label: 'Occupants informed?', value: true, notes: '' },
{ id: 'pj3', label: 'Signalling time established?', value: true, notes: '' },
{ id: 'pj4', label: 'Access/Keys available?', value: true, notes: '' },
{ id: 'pj5', label: 'Alternate plan established?', value: true, notes: '' }
],
// CONTROL: 7 ITEMS
controlRecord: [
{ id: 'cr1', label: 'Power on Indicator', value: true, notes: '' },
{ id: 'cr2', label: 'Common Trouble Lamp', value: true, notes: '' },
{ id: 'cr3', label: 'Common Trouble Signal', value: true, notes: '' },
{ id: 'cr4', label: 'AC Power Failure Trouble', value: true, notes: '' },
{ id: 'cr5', label: 'Ground Detection Lamp/Trouble', value: true, notes: '' },
{ id: 'cr6', label: 'Alarm Silence Lamp/Operation', value: true, notes: '' },
{ id: 'cr7', label: 'General Alarm Operation', value: true, notes: '' }
],
// INVENTORY: 28 ITEMS
devices: Array.from({length: 28}, (, i) => ({
id: dev${i},
deviceNum: (i + 1).toString(),
location: Zone ${Math.floor(i/10)+1} - Area ${i+1},
type: 'S',
status: 'pass',
notes: ''
})),
// EMERGENCY LIGHTS: 5 ITEMS
emergencyLights: Array.from({length: 5}, (, i) => ({
id: el${i},
unitNum: (i + 1).toString(),
unit: Emergency Unit ${i+1},
location: Expanse Area ${i+1},
cct: CCT-${i+20},
pass: true,
notes: ''
})),
// BATTERY HEALTH: 7 ITEMS (mapped internally)
batteries: {
voltageACOn: '27.1',
voltageStandby: '26.8',
voltageAlarm: '23.2',
chargingCurrent: '0.45A',
batteryAge: '2024',
physicalCondition: 'Normal',
terminalTightness: 'Secure',
notes: ''
},
completed: false,
lastUpdated: Date.now()
};
await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'inspections', baselineId), baseline);
};

const activeBuilding = inspections.find(b => b.id === selectedBuildingId);

const getDeficiencies = () => {
if (!activeBuilding) return [];
const b = activeBuilding;
const defs = [];

// Category 2: Pre-Job
b.preJob.forEach(p => { if (!p.value) defs.push({ item: p.label, id: 'PJ', cat: 'Pre-Job Checklist' }); });

// Category 3: Control Equipment
b.controlRecord.forEach(c => { if (!c.value) defs.push({ item: c.label, id: 'CR', cat: 'Control Equipment' }); });

// Category 4: Device Inventory
b.devices.forEach(d => { if (d.status === 'fail') defs.push({ item: d.location, id: d.deviceNum, cat: 'Device Inventory' }); });

// Category 5: Battery Health
if (b.batteries.physicalCondition.toLowerCase().includes('fail') || b.batteries.physicalCondition.toLowerCase().includes('poor')) {
  defs.push({ item: 'Battery Physical Condition', id: 'BAT', cat: 'Battery Health' });
}

// Category 6: Emergency Lights
b.emergencyLights.forEach(l => { if (!l.pass) defs.push({ item: l.unit, id: l.unitNum, cat: 'Emergency Lighting' }); });

return defs;


};

const updateBuilding = async (updates) => {
if (!user || !selectedBuildingId) return;
const docRef = doc(db, 'artifacts', appId, 'public', 'data', 'inspections', selectedBuildingId);
await updateDoc(docRef, { ...updates, lastUpdated: Date.now() });
};

const triggerDialogue = (config) => {
setDialogue({
isOpen: true,
title: config.title || '',
message: config.message || '',
onConfirm: config.onConfirm || null,
onCancel: config.onCancel || null,
showInput: config.showInput || false,
showTypePicker: config.showTypePicker || false,
onTypeSelect: config.onTypeSelect || null,
inputValue: config.initialValue || '',
confirmText: config.confirmText || 'Confirm',
cancelText: config.cancelText || 'Cancel'
});
};

const closeDialogue = () => setDialogue(prev => ({ ...prev, isOpen: false }));

const handleMediaUploadTrigger = (collectionName, itemId) => {
setCurrentMediaTarget({ collection: collectionName, itemId });
fileInputRef.current.click();
};

const onFileChange = (e) => {
const file = e.target.files[0];
if (!file || !currentMediaTarget || !user) return;
const reader = new FileReader();
reader.onloadend = async () => {
const base64Data = reader.result;
const fileType = file.type.startsWith('video') ? 'video' : 'image';
const { collection: coll, itemId } = currentMediaTarget;
try {
await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'media'), {
inspectionId: selectedBuildingId,
itemId: itemId,
category: coll,
type: fileType,
data: base64Data,
timestamp: Date.now()
});
} catch (err) { console.error("Failed to save media:", err); }
setCurrentMediaTarget(null);
};
reader.readAsDataURL(file);
e.target.value = null;
};

const SafeInput = ({ value, onSave, label, className, placeholder, isTextArea = false, type = "text" }) => {
const [localValue, setLocalValue] = useState(value || '');
useEffect(() => { setLocalValue(value || ''); }, [value]);
const handleBlur = () => {
if (localValue === value) return;
triggerDialogue({
title: 'Confirm Save',
message: Commit changes to "${label || 'this field'}"?,
confirmText: 'Save',
onConfirm: () => { onSave(localValue); closeDialogue(); },
onCancel: () => { setLocalValue(value); closeDialogue(); }
});
};
const Tag = isTextArea ? 'textarea' : 'input';
return (
<Tag
type={type}
className={className}
value={localValue}
placeholder={placeholder}
onChange={(e) => setLocalValue(e.target.value)}
onBlur={handleBlur}
onKeyDown={(e) => e.key === 'Enter' && !isTextArea && e.target.blur()}
/>
);
};

const handleAddRow = (category) => {
triggerDialogue({
title: Add New ${category},
message: There must be a name for it to be saved. If left blank, row will not be created.,
showInput: true,
onConfirm: (name) => {
if (!name || name.trim() === '') { closeDialogue(); return; }
triggerDialogue({
title: 'ID Tag',
message: 'Provide a numeric identifier (Optional)',
showInput: true,
onConfirm: (num) => {
if (category === 'device') {
triggerDialogue({
title: 'Select Device Type',
showTypePicker: true,
onTypeSelect: (code) => {
const newItem = { id: crypto.randomUUID(), deviceNum: num || '?', location: name, type: code, status: 'pass', notes: '' };
updateBuilding({ devices: [...activeBuilding.devices, newItem] });
closeDialogue();
}
});
} else {
const newLight = { id: crypto.randomUUID(), unitNum: num || '?', unit: name, location: 'General', cct: 'N/A', pass: true, notes: '' };
updateBuilding({ emergencyLights: [...activeBuilding.emergencyLights, newLight] });
closeDialogue();
}
}
});
}
});
};

const handleDeleteItem = (category, itemId) => {
triggerDialogue({
title: 'Confirm Deletion',
message: 'This will permanently remove this record and associated media.',
confirmText: 'Delete',
onConfirm: () => {
if (category === 'devices') updateBuilding({ devices: activeBuilding.devices.filter(d => d.id !== itemId) });
else updateBuilding({ emergencyLights: activeBuilding.emergencyLights.filter(l => l.id !== itemId) });
closeDialogue();
}
});
};

const generatePDF = async () => {
const { jsPDF } = window.jspdf;
const doc = new jsPDF();
const b = activeBuilding;
const fileName = Inspection_${b.name.replace(/\s+/g, '_')}_${activeYear}.pdf;

doc.setFontSize(22);
doc.setTextColor(30, 64, 175);
doc.text("FIRE ALARM INSPECTION REPORT", 14, 20);

doc.setFontSize(10);
doc.setTextColor(100);
doc.text(`Generated: ${new Date().toLocaleString()}`, 14, 26);

doc.autoTable({
  startY: 32,
  head: [['CATEGORY 1: BUILDING DETAILS', 'VALUE']],
  body: [
    ['Property Name', b.name],
    ['Address', b.address],
    ['Inspector', b.inspector || 'Not Assigned'],
    ['Panel Make', b.panelManufacturer],
    ['Panel Model', b.panelModel],
    ['Monitoring', b.isMonitored ? `Monitored (${b.monitoringPhone})` : 'Local Only']
  ],
  theme: 'striped',
  headStyles: { fillColor: [30, 64, 175] }
});

doc.autoTable({
  startY: doc.lastAutoTable.finalY + 10,
  head: [['CATEGORY 2: PRE-JOB CHECKLIST', 'STATUS', 'NOTES']],
  body: b.preJob.map(p => [p.label, p.value ? 'PASSED' : 'FAILED', p.notes || '-']),
  theme: 'grid',
  headStyles: { fillColor: [249, 115, 22] }
});

doc.autoTable({
  startY: doc.lastAutoTable.finalY + 10,
  head: [['CATEGORY 3: CONTROL EQUIPMENT', 'STATUS', 'NOTES']],
  body: b.controlRecord.map(c => [c.label, c.value ? 'PASSED' : 'FAILED', c.notes || '-']),
  theme: 'grid',
  headStyles: { fillColor: [79, 70, 229] }
});

doc.autoTable({
  startY: doc.lastAutoTable.finalY + 10,
  head: [['CATEGORY 4: DEVICE INVENTORY', 'ID', 'TYPE', 'LOCATION', 'STATUS']],
  body: b.devices.map(d => [d.deviceNum, d.type, d.location, d.status.toUpperCase()]),
  theme: 'striped',
  headStyles: { fillColor: [30, 41, 59] }
});

doc.autoTable({
  startY: doc.lastAutoTable.finalY + 10,
  head: [['CATEGORY 5: BATTERY HEALTH', 'READING']],
  body: [
    ['Voltage (AC On)', `${b.batteries.voltageACOn}V`],
    ['Voltage (Standby)', `${b.batteries.voltageStandby}V`],
    ['Voltage (Alarm)', `${b.batteries.voltageAlarm}V`],
    ['Charging Current', b.batteries.chargingCurrent],
    ['Battery Age/Date', b.batteries.batteryAge],
    ['Physical Condition', b.batteries.physicalCondition],
    ['Terminal Tightness', b.batteries.terminalTightness]
  ],
  theme: 'grid',
  headStyles: { fillColor: [245, 158, 11] }
});

doc.autoTable({
  startY: doc.lastAutoTable.finalY + 10,
  head: [['CATEGORY 6: EMERGENCY LIGHTS', 'NAME', 'LOCATION', 'STATUS']],
  body: b.emergencyLights.map(l => [l.unitNum, l.unit, l.location, l.pass ? 'PASSED' : 'FAILED']),
  theme: 'striped',
  headStyles: { fillColor: [5, 150, 105] }
});

const deficiencies = getDeficiencies();
doc.autoTable({
  startY: doc.lastAutoTable.finalY + 10,
  head: [['CATEGORY 7: GLOBAL DEFICIENCY SUMMARY', 'ITEM ID', 'CATEGORY']],
  body: deficiencies.length > 0 ? deficiencies.map(d => [d.item, d.id, d.cat]) : [['NO DEFICIENCIES FOUND', '', '']],
  theme: 'grid',
  headStyles: { fillColor: [220, 38, 38] }
});

doc.save(fileName);


};

const generateAndSyncWorkflow = () => {
const b = activeBuilding;
const fileNameBase = Fire_Inspection_${b.name.replace(/\s+/g, '_')}_${activeYear};

// JSON Backup File (Editable)
const editableData = {
  ...activeBuilding,
  exportDate: new Date().toISOString(),
  version: "2.6",
  mediaAssets: allMedia.filter(m => m.inspectionId === selectedBuildingId).map(m => ({ id: m.id, type: m.type, itemId: m.itemId }))
};
const jsonBlob = new Blob([JSON.stringify(editableData, null, 2)], { type: 'application/json' });

triggerDialogue({
  title: 'Generate Professional Export',
  message: 'Generating high-resolution PDF Report and editable backup file.',
  confirmText: 'Generate Files',
  onConfirm: () => {
    // Download JSON
    const jsonLink = document.createElement('a');
    jsonLink.href = URL.createObjectURL(jsonBlob);
    jsonLink.download = `${fileNameBase}_BACKUP.json`;
    jsonLink.click();

    // Download PDF after brief delay
    setTimeout(() => {
      generatePDF();
      triggerDialogue({
        title: 'Export Complete',
        message: 'Report and Backup saved. Would you like to upload them to the cloud directory now?',
        confirmText: 'Sync to Cloud',
        onConfirm: () => {
          window.open(TARGET_FOLDER_URL, '_blank');
          closeDialogue();
        }
      });
    }, 500);
  }
});


};

const MediaStrip = ({ item, category }) => {
const itemMedia = allMedia.filter(m => m.itemId === item.id);
return (
<div className="mt-3 space-y-3">
{itemMedia.length > 0 && (
<div className="flex flex-wrap gap-2 px-1">
{itemMedia.map((m) => (
<div key={m.id} className="relative group w-14 h-14 rounded-xl overflow-hidden shadow-sm border-2 border-slate-100" onClick={() => setExpandedMedia(m)}>
{m.type === 'video' ? (
<div className="w-full h-full bg-slate-800 flex items-center justify-center">
<Play size={14} className="text-white" />
<video src={m.data} className="absolute inset-0 w-full h-full object-cover opacity-40" />
</div>
) : ( <img src={m.data} className="w-full h-full object-cover" /> )}
<button onClick={(e) => {
e.stopPropagation();
triggerDialogue({
title: 'Delete Media',
message: 'Permanently remove this file?',
confirmText: 'Delete',
onConfirm: () => { deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'media', m.id)); closeDialogue(); }
});
}}
className="absolute top-0 right-0 p-0.5 bg-red-500 text-white opacity-0 group-hover:opacity-100 transition-opacity rounded-bl-lg"
><X size={10} /></button>
</div>
))}
</div>
)}
<div className="flex items-center gap-2">
<StickyNote size={12} className="text-slate-300" />
<SafeInput placeholder="Add detailed notes..." className="bg-transparent text-[10px] font-bold text-slate-500 w-full outline-none" value={item.notes} onSave={(v) => {
const updateItem = (list) => list.map(i => i.id === item.id ? { ...i, notes: v } : i);
if (category === 'preJob') updateBuilding({ preJob: updateItem(activeBuilding.preJob) });
else if (category === 'controlRecord') updateBuilding({ controlRecord: updateItem(activeBuilding.controlRecord) });
else if (category === 'devices') updateBuilding({ devices: updateItem(activeBuilding.devices) });
else if (category === 'emergencyLights') updateBuilding({ emergencyLights: updateItem(activeBuilding.emergencyLights) });
else if (category === 'batteries') updateBuilding({ batteries: { ...activeBuilding.batteries, notes: v }});
}}
/>
</div>
</div>
);
};

const CustomDialogue = () => {
if (!dialogue.isOpen) return null;
return (
<div className="fixed inset-0 z-[400] bg-slate-900/90 backdrop-blur-md flex items-center justify-center p-6">
<div className="bg-white w-full max-sm rounded-[40px] shadow-2xl overflow-hidden animate-in zoom-in-95 duration-200">
<div className="p-8 text-center">
<h3 className="text-xl font-black text-slate-800 mb-2 uppercase tracking-tight">{dialogue.title}</h3>
<p className="text-slate-500 font-bold text-xs leading-relaxed mb-6">{dialogue.message}</p>
{dialogue.showInput && (
<div className="mb-6">
<input autoFocus className="w-full bg-slate-50 p-4 rounded-2xl border-2 border-slate-100 text-center font-black outline-none focus:border-blue-400" value={dialogue.inputValue} onChange={(e) => setDialogue({...dialogue, inputValue: e.target.value})} />
</div>
)}
{dialogue.showTypePicker ? (
<div className="grid grid-cols-1 gap-2 mb-6 text-left max-h-[40vh] overflow-y-auto pr-2 custom-scrollbar">
{DEVICE_TYPES.map(type => (
<button key={type.code} onClick={() => dialogue.onTypeSelect(type.code)} className="flex items-center justify-between bg-slate-50 p-4 rounded-xl border border-slate-100 hover:bg-blue-50 transition-colors">
<span className="font-black text-blue-600 w-12 text-sm">{type.code}</span>
<span className="text-[11px] font-bold text-slate-600 flex-1">{type.label}</span>
</button>
))}
</div>
) : (
<div className="flex flex-col gap-3">
<button onClick={() => dialogue.onConfirm(dialogue.inputValue)} className="w-full py-4 rounded-2xl font-black text-sm uppercase text-white shadow-lg bg-blue-600 active:scale-95 transition-transform">{dialogue.confirmText}</button>
<button onClick={closeDialogue} className="w-full py-4 rounded-2xl font-black text-sm uppercase text-slate-400 bg-slate-50 active:scale-95 transition-transform">{dialogue.cancelText}</button>
</div>
)}
</div>
</div>
</div>
);
};

if (!user) return <div className="min-h-screen bg-slate-100 flex items-center justify-center font-black text-slate-400 animate-pulse uppercase tracking-widest">Protocol Syncing...</div>;

if (view === 'years') {
return (
<div className="min-h-screen bg-slate-100 p-6 flex flex-col items-center justify-center">
<div className="mb-8 text-center">
<div className="bg-blue-600 p-4 rounded-3xl inline-block mb-4 shadow-xl"><ShieldCheck size={48} className="text-white" /></div>
<h1 className="text-3xl font-black text-slate-800 tracking-tight uppercase">Fire-Doc V2</h1>
</div>
<div className="grid grid-cols-1 gap-4 w-full max-w-sm">
{['2024', '2025', '2026'].map(y => (
<button key={y} onClick={() => { setActiveYear(y); setView('buildings'); }} className="bg-white p-6 rounded-3xl shadow-sm flex items-center justify-between active:scale-95 transition-all">
<div className="flex items-center gap-4 font-black text-2xl text-slate-700"><Calendar className="text-blue-500" /> {y}</div>
<ChevronDown className="rotate-[-90deg] text-slate-300" />
</button>
))}
</div>
</div>
);
}

if (view === 'buildings') {
const filtered = inspections.filter(i => i.year === activeYear);
return (
<div className="min-h-screen bg-slate-50">
<CustomDialogue />
<header className="bg-blue-600 text-white p-8 pb-16 rounded-b-[48px] shadow-xl">
<button onClick={() => setView('years')} className="text-xs font-black mb-4 opacity-70 uppercase tracking-widest">Return</button>
<h1 className="text-3xl font-black tracking-tight uppercase">Properties</h1>
</header>
<div className="px-6 -mt-10 space-y-4 pb-24">
{filtered.map(b => (
<div key={b.id} onClick={() => { setSelectedBuildingId(b.id); setView('inspect'); }} className="bg-white p-6 rounded-[32px] shadow-md flex items-center justify-between active:scale-98 transition-transform">
<div className="flex items-center gap-5">
<div className={p-4 rounded-2xl ${b.completed ? 'bg-green-100 text-green-600' : 'bg-blue-50 text-blue-500'}}><Building2 size={28} /></div>
<div>
<h3 className="font-black text-lg text-slate-800 leading-tight uppercase">{b.name}</h3>
<p className="text-[10px] text-slate-400 font-black uppercase tracking-tight">Active Logs</p>
</div>
</div>
<ChevronDown className="rotate-[-90deg] text-slate-300" />
</div>
))}
</div>
</div>
);
}

if (view === 'inspect' && activeBuilding) {
const deficiencies = getDeficiencies();
return (
<div className="min-h-screen bg-slate-50 pb-44">
<CustomDialogue />
<input type="file" min-h-screen bg-slate-50 pb-44 ref={fileInputRef} className="hidden" accept="image/,video/" multiple onChange={onFileChange} />

    {expandedMedia && (
      <div className="fixed inset-0 z-[500] bg-black/98 flex items-center justify-center p-4 animate-in fade-in" onClick={() => setExpandedMedia(null)}>
        <div className="relative w-full h-full flex items-center justify-center">
          <button className="absolute top-6 right-6 text-white z-10 bg-white/10 p-2 rounded-full"><X size={28} /></button>
          {expandedMedia.type === 'video' ? (
            <video src={expandedMedia.data} controls autoPlay className="max-w-full max-h-[85vh] rounded-2xl shadow-2xl" />
          ) : ( <img src={expandedMedia.data} className="max-w-full max-h-[85vh] object-contain rounded-2xl shadow-2xl" /> )}
        </div>
      </div>
    )}

    <header className="bg-white border-b sticky top-0 z-20 p-5 flex items-center justify-between shadow-sm">
      <button onClick={() => setView('buildings')} className="text-blue-600 font-black text-[10px] uppercase">Exit Session</button>
      <div className="text-center">
        <h2 className="font-black text-slate-800 text-[11px] uppercase truncate max-w-[150px]">{activeBuilding.name}</h2>
        <p className="text-[8px] text-slate-400 font-black uppercase tracking-widest">Inspection Live</p>
      </div>
      <button onClick={() => updateBuilding({ completed: true })} className="bg-blue-600 text-white p-2 rounded-xl shadow-lg"><Save size={16} /></button>
    </header>

    <div className="p-4 space-y-4">
      {/* CATEGORY 1: DETAILS (6 Items) */}
      <div className="rounded-[32px] shadow-sm border border-blue-100 overflow-hidden bg-white">
        <button onClick={() => toggleSection('details')} className="w-full flex items-center justify-between p-6 bg-blue-600 text-white">
          <span className="font-black text-[11px] uppercase tracking-[0.15em]">1. Building Details</span>
          {openSections.details ? <ChevronUp size={20} /> : <ChevronDown size={20} />}
        </button>
        {openSections.details && (
          <div className="p-6 space-y-5">
            {[
              { label: "1. Inspector", key: "inspector", placeholder: "Blank - Set Inspector Name" },
              { label: "2. Address", key: "address" },
              { label: "3. Panel Make", key: "panelManufacturer" },
              { label: "4. Panel Model", key: "panelModel" }
            ].map(f => (
              <div key={f.key} className="space-y-1">
                <label className="text-[9px] font-black text-slate-400 uppercase">{f.label}</label>
                <SafeInput label={f.label} className="w-full bg-slate-50 p-4 rounded-2xl text-sm font-black outline-none border-2 border-transparent focus:border-blue-200" value={activeBuilding[f.key]} onSave={(val) => updateBuilding({ [f.key]: val })} placeholder={f.placeholder} />
              </div>
            ))}
            <div className="space-y-1">
              <label className="text-[9px] font-black text-slate-400 uppercase">5. Monitoring Status</label>
              <button onClick={() => {
                  const next = !activeBuilding.isMonitored;
                  triggerDialogue({
                      title: 'Status Update',
                      message: `Set system to ${next ? 'Monitored' : 'Local Only'}?`,
                      confirmText: 'Confirm',
                      onConfirm: () => { updateBuilding({ isMonitored: next }); closeDialogue(); }
                  });
              }} className={`w-full p-4 rounded-2xl text-[10px] font-black uppercase flex items-center justify-center gap-2 transition-colors ${activeBuilding.isMonitored ? 'bg-indigo-600 text-white' : 'bg-slate-100 text-slate-400'}`}>
                {activeBuilding.isMonitored ? <Eye size={14}/> : <EyeOff size={14}/>}
                {activeBuilding.isMonitored ? 'System Monitored' : 'Local Only'}
              </button>
            </div>
            {activeBuilding.isMonitored && (
              <div className="space-y-1 animate-in slide-in-from-top-2 duration-200">
                <label className="text-[9px] font-black text-slate-400 uppercase">6. Monitoring Phone</label>
                <SafeInput label="Phone" className="w-full bg-slate-50 p-4 rounded-2xl text-sm font-black outline-none border-2 border-indigo-100" value={activeBuilding.monitoringPhone} onSave={(val) => updateBuilding({ monitoringPhone: val })} />
              </div>
            )}
          </div>
        )}
      </div>

      {[
        { id: 'prejob', title: '2. Pre-Job Checklist', data: activeBuilding.preJob, color: 'bg-orange-500', category: 'preJob' },
        { id: 'control', title: '3. Control Equipment', data: activeBuilding.controlRecord, color: 'bg-indigo-600', category: 'controlRecord' },
        { id: 'inventory', title: '4. Device Inventory', data: activeBuilding.devices, color: 'bg-slate-800', category: 'devices', canAdd: true },
        { id: 'emergencyLights', title: '6. Emergency Lights', data: activeBuilding.emergencyLights, color: 'bg-emerald-600', category: 'emergencyLights', canAdd: true }
      ].map(sec => (
        <div key={sec.id} className={`rounded-[32px] shadow-sm border border-slate-100 overflow-hidden bg-white`}>
          <button onClick={() => toggleSection(sec.id)} className={`w-full flex items-center justify-between p-6 ${sec.color} text-white`}>
            <span className="font-black text-[11px] uppercase tracking-[0.15em]">{sec.title} ({sec.data.length} Items)</span>
            {openSections[sec.id] ? <ChevronUp size={20} /> : <ChevronDown size={20} />}
          </button>
          {openSections[sec.id] && (
            <div className="p-4 space-y-4">
              {sec.data.map((item, idx) => (
                <div key={item.id} className="p-5 rounded-[24px] bg-slate-50 border border-slate-100">
                  <div className="flex items-start justify-between gap-3 mb-2">
                    <div className="flex-1 space-y-2">
                      <div className="flex items-center gap-2">
                         <span className="text-[10px] font-black text-slate-300">{idx+1}</span>
                         <SafeInput label="Field Name" className="bg-transparent font-black text-[11px] uppercase text-slate-800 outline-none w-full" value={item.label || item.location || item.unit} onSave={(v) => {
                             const updateField = (list) => list.map(i => i.id === item.id ? { ...i, [item.label ? 'label' : (item.location ? 'location' : 'unit')]: v } : i);
                             updateBuilding({ [sec.category]: updateField(sec.data) });
                           }} />
                      </div>
                      <div className="flex items-center gap-3">
                        {(item.deviceNum !== undefined || item.unitNum !== undefined) && (
                          <div className="flex items-center gap-1 bg-white px-2 py-1 rounded-lg border border-slate-100">
                            <span className="text-[8px] font-black text-slate-300">#</span>
                            <SafeInput className="text-[10px] font-black text-slate-600 outline-none w-8" value={item.deviceNum || item.unitNum} onSave={(v) => {
                                const updateId = (list) => list.map(i => i.id === item.id ? { ...i, [item.deviceNum !== undefined ? 'deviceNum' : 'unitNum']: v } : i);
                                updateBuilding({ [sec.category]: updateId(sec.data) });
                              }} />
                          </div>
                        )}
                        {item.type && (
                          <button onClick={() => triggerDialogue({
                            title: 'Update Device Type',
                            showTypePicker: true,
                            onTypeSelect: (code) => {
                              const updateType = (list) => list.map(i => i.id === item.id ? { ...i, type: code } : i);
                              updateBuilding({ [sec.category]: updateType(sec.data) });
                              closeDialogue();
                            }
                          })} className="text-[8px] font-black text-blue-500 bg-blue-50 px-2 py-1 rounded uppercase flex items-center gap-1">
                            {item.type} <ChevronDown size={8}/>
                          </button>
                        )}
                      </div>
                    </div>
                    <div className="flex items-center gap-2">
                      <button onClick={() => handleMediaUploadTrigger(sec.category, item.id)} className="p-2 text-slate-400 hover:text-blue-500"><Camera size={18} /></button>
                      <button onClick={() => {
                        const updateValue = (list) => list.map(i => i.id === item.id ? { ...i, [sec.id === 'emergencyLights' ? 'pass' : (sec.id === 'inventory' ? 'status' : 'value')]: sec.id === 'inventory' ? (i.status === 'pass' ? 'fail' : 'pass') : !i[sec.id === 'emergencyLights' ? 'pass' : 'value'] } : i);
                        updateBuilding({ [sec.category]: updateValue(sec.data) });
                      }} className={`w-8 h-8 rounded-full flex items-center justify-center transition-all ${ (item.value || item.pass || item.status === 'pass') ? 'bg-green-500' : 'bg-red-500' } text-white`}>
                        {(item.value || item.pass || item.status === 'pass') ? <Check size={16} /> : <X size={16} />}
                      </button>
                      {sec.canAdd && <button onClick={() => handleDeleteItem(sec.category, item.id)} className="p-2 text-slate-200 hover:text-red-400"><Trash2 size={16}/></button>}
                    </div>
                  </div>
                  <MediaStrip item={item} category="sec.category" />
                </div>
              ))}
              {sec.canAdd && <button onClick={() => handleAddRow(sec.id === 'inventory' ? 'device' : 'light')} className="w-full py-8 border-4 border-dotted border-slate-100 rounded-[32px] text-slate-300 font-black uppercase text-[10px] hover:bg-slate-50 active:scale-98">+ APPEND NEW RECORD</button>}
            </div>
          )}
        </div>
      ))}

      {/* CATEGORY 5: BATTERY HEALTH (7 Items) */}
      <div className="rounded-[32px] shadow-sm border border-amber-100 overflow-hidden bg-white">
        <button onClick={() => toggleSection('batteries')} className="w-full flex items-center justify-between p-6 bg-amber-500 text-white">
          <span className="font-black text-[11px] uppercase tracking-[0.15em]">5. Battery Health (7 Items)</span>
          {openSections.batteries ? <ChevronUp size={20} /> : <ChevronDown size={20} />}
        </button>
        {openSections.batteries && (
          <div className="p-6 space-y-4">
            <div className="grid grid-cols-2 gap-4">
              {[
                { label: "1. Standby (V)", k: "voltageStandby" }, { label: "2. Alarm (V)", k: "voltageAlarm" },
                { label: "3. AC On (V)", k: "voltageACOn" }, { label: "4. Charge (A)", k: "chargingCurrent" }
              ].map(b => (
                <div key={b.k} className="space-y-1">
                  <label className="text-[9px] font-black text-slate-400 uppercase">{b.label}</label>
                  <SafeInput label={b.label} className="w-full bg-slate-50 p-4 rounded-2xl text-sm font-black outline-none" value={activeBuilding.batteries[b.k]} onSave={(v) => updateBuilding({ batteries: { ...activeBuilding.batteries, [b.k]: v }})} />
                </div>
              ))}
            </div>
            {[
              { label: "5. Battery Age/Date", k: "batteryAge" }, { label: "6. Physical Condition", k: "physicalCondition" },
              { label: "7. Terminal Tightness", k: "terminalTightness" }
            ].map(b => (
              <div key={b.k} className="space-y-1">
                <label className="text-[9px] font-black text-slate-400 uppercase">{b.label}</label>
                <SafeInput label={b.label} className="w-full bg-slate-50 p-4 rounded-2xl text-sm font-black outline-none" value={activeBuilding.batteries[b.k]} onSave={(v) => updateBuilding({ batteries: { ...activeBuilding.batteries, [b.k]: v }})} />
              </div>
            ))}
            <MediaStrip item={{...activeBuilding.batteries, id: activeBuilding.id}} category="batteries" />
          </div>
        )}
      </div>

      {/* CATEGORY 7: DEFICIENCY SUMMARY */}
      <div className="rounded-[32px] shadow-sm border border-red-100 overflow-hidden bg-white">
         <button onClick={() => toggleSection('summary')} className="w-full flex items-center justify-between p-6 bg-red-600 text-white font-black text-[11px] uppercase tracking-[0.15em]">
           7. Deficiency Summary
           {openSections.summary ? <ChevronUp size={20} /> : <ChevronDown size={20} />}
         </button>
         {openSections.summary && (
           <div className="p-6 space-y-4">
             {deficiencies.length > 0 ? (
               deficiencies.map((def, i) => (
                <div key={i} className="p-5 bg-red-50 rounded-2xl border-2 border-red-100">
                  <div className="flex justify-between items-start mb-2">
                    <span className="text-[10px] font-black text-red-600 uppercase tracking-tighter">Category: {def.cat}</span>
                    <span className="text-[10px] font-black text-slate-400">ID: {def.id}</span>
                  </div>
                  <p className="text-xs font-black text-slate-700 uppercase">{def.item}</p>
                </div>
              ))
             ) : (
               <p className="text-center py-6 text-[10px] font-black text-slate-400 uppercase">All systems within parameters</p>
             )}
           </div>
         )}
      </div>

      {/* FINAL ACTIONS */}
      <div className="pt-10">
        <button onClick={generateAndSyncWorkflow} className="w-full flex flex-col items-center justify-center gap-2 bg-blue-600 p-8 rounded-[40px] text-white shadow-2xl active:scale-95 transition-transform">
           <div className="flex items-center gap-4"><FileDown size={28} /><span className="text-[13px] font-black uppercase tracking-widest">Generate & Cloud Sync</span></div>
           <p className="text-[8px] font-black opacity-60 uppercase tracking-widest">PRO PDF REPORT (1-7) + EDITABLE DATA</p>
        </button>
      </div>
    </div>
  </div>
);


}
return null;
};

export default App;