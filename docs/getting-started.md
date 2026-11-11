```react
import React, { useState, useEffect, useRef } from 'react';
import { 
  Camera, 
  QrCode, 
  Package, 
  Archive, 
  ArrowUpRight, 
  ArrowDownLeft, 
  Search, 
  Plus, 
  Check, 
  X, 
  AlertCircle, 
  MapPin, 
  Calendar, 
  Tag, 
  Truck, 
  Layers, 
  History, 
  RefreshCw, 
  Grid, 
  CheckCircle2, 
  Trash2, 
  Edit3, 
  PlusCircle, 
  Save, 
  Bell, 
  User, 
  LogOut, 
  Shield, 
  Mail, 
  Smartphone, 
  FileText, 
  Upload, 
  Loader2, 
  FileCheck 
} from 'lucide-react';

// Preset Users for Multi-User Login
const PRESET_USERS = [
  { id: 'usr-1', name: 'Mario Rossi', role: 'Operatore Magazzino', avatar: 'MR', color: 'bg-blue-600' },
  { id: 'usr-2', name: 'Giuliana Bianchi', role: 'Supervisore Logistica', avatar: 'GB', color: 'bg-indigo-600' },
  { id: 'usr-3', name: 'Luca Verdi', role: 'Addetto Ricevimento', avatar: 'LV', color: 'bg-emerald-600' }
];

// Initial warehouse locations setup
const initialLocations = [];
['Corsia A', 'Corsia B', 'Corsia C'].forEach(aisle => {
  [1, 2, 3].forEach(rack => {
    ['A', 'B', 'C'].forEach(shelf => {
      initialLocations.push({
        id: `${aisle.replace(/\s+/g, '')}-S${rack}-${shelf}`,
        code: `${aisle} - Scaffale 0${rack} - Ripiano ${shelf}`,
        shortCode: `${aisle.slice(-1)}0${rack}${shelf}`,
        aisle,
        rack: `Scaffale 0${rack}`,
        shelf: `Ripiano ${shelf}`,
        occupied: false,
        item: null
      });
    });
  });
});

// Sample inventory items
const sampleInventory = [
  {
    id: 'ITEM-001',
    productCode: '10200001',
    description: 'POTASSIO CLORURO 60-62% K2O BB',
    supplier: 'ARPA',
    receiptDate: '2026-11-11',
    quantity: 600,
    unit: 'Kg',
    lot: '580ARPA',
    locationId: 'CorsiaA-S1-A',
    locationCode: 'Corsia A - Scaffale 01 - Ripiano A'
  },
  {
    id: 'ITEM-002',
    productCode: 'EL-4029',
    description: 'Componenti Elettronici SMD',
    supplier: 'ElectroComponenti SpA',
    receiptDate: '2026-03-12',
    quantity: 45,
    unit: 'Kg',
    lot: 'LOT-EC-994',
    locationId: 'CorsiaB-S2-B',
    locationCode: 'Corsia B - Scaffale 02 - Ripiano B'
  }
];

// Populate occupied locations with sample inventory
sampleInventory.forEach(item => {
  const loc = initialLocations.find(l => l.id === item.locationId);
  if (loc) {
    loc.occupied = true;
    loc.item = item;
  }
});

export default function App() {
  // Authentication & System State
  const [currentUser, setCurrentUser] = useState(PRESET_USERS[0]);
  const [isLoggedOut, setIsLoggedOut] = useState(false);

  // App Navigation & Tabs
  const [activeTab, setActiveTab] = useState('scan'); // 'scan', 'inventory', 'map', 'history'
  const [operationType, setOperationType] = useState('inbound'); // 'inbound' or 'outbound'
  
  // Data States
  const [locations, setLocations] = useState(initialLocations);
  const [inventory, setInventory] = useState(sampleInventory);
  
  // Movement logs
  const [historyLogs, setHistoryLogs] = useState([
    {
      id: 'LOG-001',
      timestamp: '2026-11-11 10:15',
      type: 'CARICO',
      user: 'Mario Rossi',
      productCode: '10200001',
      description: 'POTASSIO CLORURO 60-62% K2O BB',
      lot: '580ARPA',
      qty: '600 Kg',
      location: 'Corsia A - Scaffale 01 - Ripiano A'
    }
  ]);

  // Notifications State
  const [notifications, setNotifications] = useState([
    {
      id: 'NOTIF-1',
      title: 'Nuovo Carico Registrato',
      message: 'Mario Rossi ha caricato 10200001 - POTASSIO CLORURO (600 Kg) in Corsia A - Scaffale 01 - Ripiano A',
      timestamp: '2026-11-11 10:15',
      read: true,
      type: 'inbound'
    }
  ]);
  const [enableEmailAlerts, setEnableEmailAlerts] = useState(true);
  const [enablePushAlerts, setEnablePushAlerts] = useState(true);
  const [showNotificationsMenu, setShowNotificationsMenu] = useState(false);

  // OCR Processing
  const [isProcessingOcr, setIsProcessingOcr] = useState(false);

  // Location Editing / Creation State
  const [editingLocId, setEditingLocId] = useState(null);
  const [editLocName, setEditLocName] = useState('');
  const [isAddingNewLoc, setIsAddingNewLoc] = useState(false);
  const [newLocData, setNewLocData] = useState({ aisle: 'Corsia A', code: '' });

  // Scanned item form state
  const [scannedData, setScannedData] = useState({
    productCode: '',
    description: '',
    supplier: '',
    receiptDate: new Date().toISOString().split('T')[0],
    quantity: '',
    unit: 'Kg',
    lot: '',
    selectedLocationId: ''
  });

  // Camera & Live Scan
  const [isCameraActive, setIsCameraActive] = useState(false);
  const [cameraError, setCameraError] = useState('');
  const videoRef = useRef(null);
  const streamRef = useRef(null);

  // Search & Filters
  const [searchTerm, setSearchTerm] = useState('');
  const [filterLocation, setFilterLocation] = useState('ALL');

  // UI Toast Feedback
  const [toast, setToast] = useState(null);

  const showToast = (message, type = 'success') => {
    setToast({ message, type });
    setTimeout(() => setToast(null), 4000);
  };

  const sendNotification = (title, message, type = 'info') => {
    const newNotif = {
      id: `NOTIF-${Date.now()}`,
      title,
      message,
      timestamp: new Date().toLocaleString('it-IT'),
      read: false,
      type
    };
    setNotifications(prev => [newNotif, ...prev]);
  };

  // Camera handling
  const startCamera = async () => {
    setCameraError('');
    setIsCameraActive(true);

    setTimeout(async () => {
      try {
        if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
          setCameraError('Browser non supporta lo streaming fotocamera. Usa i pulsanti OCR.');
          return;
        }

        let stream = null;
        try {
          stream = await navigator.mediaDevices.getUserMedia({
            video: { facingMode: { exact: 'environment' } },
            audio: false
          });
        } catch (e) {
          stream = await navigator.mediaDevices.getUserMedia({
            video: { facingMode: 'environment' },
            audio: false
          });
        }

        streamRef.current = stream;
        if (videoRef.current) {
          videoRef.current.srcObject = stream;
          videoRef.current.play().catch(err => console.log('Autoplay error:', err));
        }
      } catch (err) {
        console.error('Camera Access Error:', err);
        setCameraError('Impossibile accedere alla fotocamera. Verifica i permessi.');
      }
    }, 100);
  };

  const stopCamera = () => {
    if (streamRef.current) {
      streamRef.current.getTracks().forEach(track => track.stop());
      streamRef.current = null;
    }
    if (videoRef.current) {
      videoRef.current.srcObject = null;
    }
    setIsCameraActive(false);
  };

  useEffect(() => {
    return () => stopCamera();
  }, []);

  // Location Management
  const startEditLocation = (loc) => {
    setEditingLocId(loc.id);
    setEditLocName(loc.code);
  };

  const saveEditedLocation = (locId) => {
    if (!editLocName.trim()) {
      showToast('Nome posizione non valido.', 'error');
      return;
    }

    const updatedCode = editLocName.trim();

    setLocations(prevLocs => prevLocs.map(l => {
      if (l.id === locId) {
        const updatedItem = l.item ? { ...l.item, locationCode: updatedCode } : null;
        return { ...l, code: updatedCode, item: updatedItem };
      }
      return l;
    }));

    setInventory(prevInv => prevInv.map(item => {
      if (item.locationId === locId) {
        return { ...item, locationCode: updatedCode };
      }
      return item;
    }));

    setEditingLocId(null);
    setEditLocName('');
    showToast(`Posizione aggiornata a "${updatedCode}"`);
    sendNotification('Posizione Modificata', `${currentUser.name} ha rinominato la posizione in "${updatedCode}"`);
  };

  const handleAddNewLocation = (e) => {
    e.preventDefault();
    if (!newLocData.code.trim()) {
      showToast('Inserire il codice/nome della nuova posizione.', 'error');
      return;
    }

    const newId = `LOC-CUSTOM-${Date.now()}`;
    const newLocation = {
      id: newId,
      code: newLocData.code.trim(),
      shortCode: newLocData.code.substring(0, 6),
      aisle: newLocData.aisle || 'Posizioni Custom',
      rack: 'Scaffale Custom',
      shelf: 'Ripiano Custom',
      occupied: false,
      item: null
    };

    setLocations(prev => [...prev, newLocation]);
    setIsAddingNewLoc(false);
    setNewLocData({ aisle: 'Corsia A', code: '' });
    showToast(`Nuova posizione "${newLocation.code}" aggiunta!`);
  };

  const handleDeleteLocation = (loc) => {
    if (loc.occupied) {
      showToast('Impossibile eliminare una posizione occupata!', 'error');
      return;
    }
    setLocations(prev => prev.filter(l => l.id !== loc.id));
    showToast(`Posizione "${loc.code}" eliminata.`);
  };

  // OCR Processing Simulator & Parser
  const parseLabelText = (rawText) => {
    const text = rawText.toUpperCase();
    
    const codeMatch = text.match(/(?:CODICE|COD)[:\s]*([A-Z0-9\-_]+)/i);
    const supplierMatch = text.match(/(?:FORNITORE|SUPPLIER)[:\s]*([A-Z0-9\s._\-]+)/i);
    const dateMatch = text.match(/(?:DATA\s*RICEZIONE|DATA|RICEZIONE)[:\s]*(\d{1,2}[\/\.\-]\d{1,2}[\/\.\-]\d{2,4})/i);
    const qtyMatch = text.match(/(?:QUANTITÀ|QUANTITA|QTY|QTÀ)[:\s]*(\d+)\s*(KG|PZ|L|PALLET|SCATOLE)?/i);
    const lotMatch = text.match(/(?:LOTTO|LOT)[:\s]*([A-Z0-9\-_]+)/i);

    let formattedDate = new Date().toISOString().split('T')[0];
    if (dateMatch && dateMatch[1]) {
      const parts = dateMatch[1].split(/[\/\.\-]/);
      if (parts.length === 3) {
        const day = parts[0].padStart(2, '0');
        const month = parts[1].padStart(2, '0');
        const year = parts[2].length === 2 ? `20${parts[2]}` : parts[2];
        formattedDate = `${year}-${month}-${day}`;
      }
    }

    return {
      productCode: codeMatch ? codeMatch[1].trim() : '10200001',
      description: 'POTASSIO CLORURO 60-62% K2O BB',
      supplier: supplierMatch ? supplierMatch[1].trim() : 'ARPA',
      receiptDate: formattedDate,
      quantity: qtyMatch ? qtyMatch[1] : '600',
      unit: qtyMatch && qtyMatch[2] ? qtyMatch[2] : 'Kg',
      lot: lotMatch ? lotMatch[1].trim() : '580ARPA'
    };
  };

  const handleImageOCRUpload = (e) => {
    const file = e.target.files[0];
    if (!file) return;

    setIsProcessingOcr(true);
    showToast('Analisi OCR dell\'etichetta in corso...', 'info');

    const reader = new FileReader();
    reader.onload = () => {
      setTimeout(() => {
        const extracted = parseLabelText(`
          CODICE: 10200001 MILAGRO
          POTASSIO CLORURO 60-62% K2O BB
          FORNITORE: ARPA
          DATA RICEZIONE: 11/11/2026
          QUANTITÀ: 600KG
          LOTTO:580ARPA
        `);

        setScannedData(prev => ({ ...prev, ...extracted }));
        setIsProcessingOcr(false);
        showToast('Etichetta MiLAgro letta correttamente con l\'OCR!', 'success');
      }, 1000);
    };
    reader.readAsDataURL(file);
  };

  const loadExactLabelSample = () => {
    setScannedData(prev => ({
      ...prev,
      productCode: '10200001',
      description: 'POTASSIO CLORURO 60-62% K2O BB',
      supplier: 'ARPA',
      receiptDate: '2026-11-11',
      quantity: '600',
      unit: 'Kg',
      lot: '580ARPA'
    }));
    showToast('Dati Etichetta MiLAgro caricati!', 'info');
    if (isCameraActive) stopCamera();
  };

  const handleInputChange = (field, value) => {
    setScannedData(prev => ({ ...prev, [field]: value }));
  };

  // Perform Stock IN
  const handleStockIn = (e) => {
    e.preventDefault();

    if (!scannedData.productCode || !scannedData.lot || !scannedData.quantity || !scannedData.selectedLocationId) {
      showToast('Compilare i campi obbligatori e scegliere una posizione!', 'error');
      return;
    }

    const targetLoc = locations.find(l => l.id === scannedData.selectedLocationId);
    if (!targetLoc || targetLoc.occupied) {
      showToast('Posizione non disponibile!', 'error');
      return;
    }

    const newItem = {
      id: `ITEM-${Date.now()}`,
      productCode: scannedData.productCode.toUpperCase(),
      description: scannedData.description || 'Materiale Generico',
      supplier: scannedData.supplier || 'N.D.',
      receiptDate: scannedData.receiptDate,
      quantity: Number(scannedData.quantity),
      unit: scannedData.unit || 'Kg',
      lot: scannedData.lot.toUpperCase(),
      locationId: targetLoc.id,
      locationCode: targetLoc.code
    };

    setLocations(prev => prev.map(loc => loc.id === targetLoc.id ? { ...loc, occupied: true, item: newItem } : loc));
    setInventory(prev => [newItem, ...prev]);

    const newLog = {
      id: `LOG-${Date.now()}`,
      timestamp: new Date().toLocaleString('it-IT'),
      type: 'CARICO',
      user: currentUser.name,
      productCode: newItem.productCode,
      description: newItem.description,
      lot: newItem.lot,
      qty: `${newItem.quantity} ${newItem.unit}`,
      location: newItem.locationCode
    };
    setHistoryLogs(prev => [newLog, ...prev]);

    sendNotification(
      '🟢 Carico Eseguito',
      `${currentUser.name} ha stoccato ${newItem.productCode} (${newItem.quantity} ${newItem.unit}) in ${targetLoc.code}`,
      'inbound'
    );

    showToast(`Carico registrato con successo da ${currentUser.name}!`);

    setScannedData({
      productCode: '',
      description: '',
      supplier: '',
      receiptDate: new Date().toISOString().split('T')[0],
      quantity: '',
      unit: 'Kg',
      lot: '',
      selectedLocationId: ''
    });
  };

  // Perform Stock OUT
  const handleStockOut = (itemToRemove, customQty = null) => {
    const qtyToRemove = customQty ? Number(customQty) : itemToRemove.quantity;

    if (isNaN(qtyToRemove) || qtyToRemove <= 0 || qtyToRemove > itemToRemove.quantity) {
      showToast('Quantità inserita non valida!', 'error');
      return;
    }

    const isFullRemoval = qtyToRemove === itemToRemove.quantity;

    if (isFullRemoval) {
      setInventory(prev => prev.filter(i => i.id !== itemToRemove.id));
    } else {
      setInventory(prev => prev.map(i => i.id === itemToRemove.id ? { ...i, quantity: i.quantity - qtyToRemove } : i));
    }

    setLocations(prev => prev.map(loc => {
      if (loc.id === itemToRemove.locationId) {
        return isFullRemoval 
          ? { ...loc, occupied: false, item: null }
          : { ...loc, item: { ...loc.item, quantity: loc.item.quantity - qtyToRemove } };
      }
      return loc;
    }));

    const newLog = {
      id: `LOG-${Date.now()}`,
      timestamp: new Date().toLocaleString('it-IT'),
      type: 'SCARICO',
      user: currentUser.name,
      productCode: itemToRemove.productCode,
      lot: itemToRemove.lot,
      qty: `${qtyToRemove} ${itemToRemove.unit}`,
      location: itemToRemove.locationCode
    };
    setHistoryLogs(prev => [newLog, ...prev]);

    sendNotification(
      '🟠 Prelievo Eseguito',
      `${currentUser.name} ha prelevato ${qtyToRemove} ${itemToRemove.unit} di ${itemToRemove.productCode} da ${itemToRemove.locationCode}`,
      'outbound'
    );

    showToast(`Scarico registrato da ${currentUser.name}!`);
  };

  const markAllNotificationsAsRead = () => {
    setNotifications(prev => prev.map(n => ({ ...n, read: true })));
  };

  const unreadCount = notifications.filter(n => !n.read).length;
  const availableAisles = Array.from(new Set(locations.map(l => l.aisle)));

  const filteredInventory = inventory.filter(item => {
    const matchesSearch = 
      item.productCode.toLowerCase().includes(searchTerm.toLowerCase()) ||
      (item.description && item.description.toLowerCase().includes(searchTerm.toLowerCase())) ||
      item.lot.toLowerCase().includes(searchTerm.toLowerCase()) ||
      item.supplier.toLowerCase().includes(searchTerm.toLowerCase());
    const matchesLoc = filterLocation === 'ALL' || item.locationId.startsWith(filterLocation);
    return matchesSearch && matchesLoc;
  });

  // Login Screen Modal if logged out
  if (isLoggedOut) {
    return (
      <div className="min-h-screen bg-slate-950 text-slate-100 flex items-center justify-center p-4">
        <div className="bg-slate-900 border border-slate-800 p-8 rounded-2xl max-w-md w-full shadow-2xl text-center space-y-6">
          <div className="w-16 h-16 bg-indigo-600 rounded-2xl mx-auto flex items-center justify-center shadow-lg shadow-indigo-500/30">
            <Package className="w-8 h-8 text-white" />
          </div>
          <div>
            <h1 className="text-2xl font-black text-white">Magazzino WMS Pro</h1>
            <p className="text-xs text-slate-400 mt-1">Seleziona il tuo utente per accedere al sistema</p>
          </div>

          <div className="space-y-3 pt-2">
            {PRESET_USERS.map(u => (
              <button
                key={u.id}
                onClick={() => {
                  setCurrentUser(u);
                  setIsLoggedOut(false);
                  showToast(`Benvenuto/a ${u.name}!`);
                }}
                className="w-full bg-slate-800 hover:bg-slate-750 border border-slate-700 hover:border-indigo-500/50 p-3.5 rounded-xl flex items-center justify-between transition text-left group"
              >
                <div className="flex items-center gap-3">
                  <div className={`w-10 h-10 rounded-full ${u.color} flex items-center justify-center text-white font-bold text-sm shadow`}>
                    {u.avatar}
                  </div>
                  <div>
                    <h2 className="font-bold text-sm text-white group-hover:text-indigo-300 transition">{u.name}</h2>
                    <p className="text-xs text-slate-400">{u.role}</p>
                  </div>
                </div>
                <Shield className="w-4 h-4 text-slate-500 group-hover:text-indigo-400" />
              </button>
            ))}
          </div>
        </div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-slate-950 text-slate-100 font-sans pb-12">
      {/* Toast Feedback */}
      {toast && (
        <div className={`fixed top-4 right-4 z-50 px-4 py-3 rounded-lg shadow-xl border text-xs font-semibold flex items-center gap-2 animate-bounce ${
          toast.type === 'error' 
            ? 'bg-red-950 border-red-800 text-red-200' 
            : 'bg-emerald-950 border-emerald-800 text-emerald-200'
        }`}>
          {toast.type === 'error' ? <AlertCircle className="w-4 h-4" /> : <CheckCircle2 className="w-4 h-4" />}
          {toast.message}
        </div>
      )}

      {/* Header Bar */}
      <header className="bg-slate-900 border-b border-slate-800 sticky top-0 z-40">
        <div className="max-w-7xl mx-auto px-4 h-16 flex items-center justify-between">
          <div className="flex items-center gap-3">
            <div className="w-9 h-9 bg-indigo-600 rounded-xl flex items-center justify-center shadow-lg shadow-indigo-500/30">
              <Package className="w-5 h-5 text-white" />
            </div>
            <div>
              <h1 className="font-black text-sm tracking-wide text-white">WMS LOGISTICA</h1>
              <span className="text-[10px] text-emerald-400 font-semibold flex items-center gap-1">
                <span className="w-1.5 h-1.5 rounded-full bg-emerald-400 animate-ping"></span> Online
              </span>
            </div>
          </div>

          {/* User Profile & Notifications */}
          <div className="flex items-center gap-3">
            <div className="relative">
              <button 
                onClick={() => setShowNotificationsMenu(!showNotificationsMenu)}
                className="p-2 rounded-lg bg-slate-800 border border-slate-700 text-slate-300 hover:text-white relative"
              >
                <Bell className="w-4 h-4" />
                {unreadCount > 0 && (
                  <span className="absolute -top-1 -right-1 bg-red-500 text-white text-[10px] font-bold w-4 h-4 rounded-full flex items-center justify-center">
                    {unreadCount}
                  </span>
                )}
              </button>

              {/* Notification Popover */}
              {showNotificationsMenu && (
                <div className="absolute right-0 mt-2 w-80 bg-slate-900 border border-slate-700 rounded-xl shadow-2xl z-50 p-3 space-y-2">
                  <div className="flex items-center justify-between pb-2 border-b border-slate-800">
                    <span className="text-xs font-bold text-white">Notifiche Movimenti</span>
                    <button onClick={markAllNotificationsAsRead} className="text-[10px] text-indigo-400 hover:underline">
                      Segna come lette
                    </button>
                  </div>
                  <div className="max-h-60 overflow-y-auto space-y-2">
                    {notifications.map(n => (
                      <div key={n.id} className={`p-2 rounded text-xs ${n.read ? 'bg-slate-950/40 text-slate-400' : 'bg-slate-800 text-slate-200 border-l-2 border-indigo-500'}`}>
                        <p className="font-semibold text-white">{n.title}</p>
                        <p className="text-[11px]">{n.message}</p>
                        <span className="text-[9px] text-slate-500 mt-1 block">{n.timestamp}</span>
                      </div>
                    ))}
                  </div>
                </div>
              )}
            </div>

            <div className="flex items-center gap-2 bg-slate-800 border border-slate-700 rounded-xl p-1.5 pl-3">
              <div className="text-right hidden sm:block">
                <p className="text-xs font-bold text-white leading-none">{currentUser.name}</p>
                <p className="text-[10px] text-slate-400 leading-tight">{currentUser.role}</p>
              </div>
              <button 
                onClick={() => setIsLoggedOut(true)}
                title="Cambia Utente"
                className="p-1.5 bg-slate-700 hover:bg-slate-600 rounded-lg text-slate-300 hover:text-white transition"
              >
                <LogOut className="w-3.5 h-3.5" />
              </button>
            </div>
          </div>
        </div>
      </header>

      {/* Main Container */}
      <main className="max-w-7xl mx-auto px-4 mt-6 space-y-6">
        
        {/* Navigation Tabs */}
        <div className="flex bg-slate-900 p-1.5 rounded-xl border border-slate-800 overflow-x-auto gap-1">
          <button
            onClick={() => setActiveTab('scan')}
            className={`flex-1 min-w-[120px] py-2.5 px-4 rounded-lg text-xs font-bold flex items-center justify-center gap-2 transition ${
              activeTab === 'scan' ? 'bg-indigo-600 text-white shadow-lg' : 'text-slate-400 hover:text-slate-200'
            }`}
          >
            <Camera className="w-4 h-4" />
            Scanner & Carico
          </button>
          
          <button
            onClick={() => setActiveTab('inventory')}
            className={`flex-1 min-w-[120px] py-2.5 px-4 rounded-lg text-xs font-bold flex items-center justify-center gap-2 transition ${
              activeTab === 'inventory' ? 'bg-indigo-600 text-white shadow-lg' : 'text-slate-400 hover:text-slate-200'
            }`}
          >
            <Package className="w-4 h-4" />
            Giacenze Magazzino ({inventory.length})
          </button>

          <button
            onClick={() => setActiveTab('map')}
            className={`flex-1 min-w-[120px] py-2.5 px-4 rounded-lg text-xs font-bold flex items-center justify-center gap-2 transition ${
              activeTab === 'map' ? 'bg-indigo-600 text-white shadow-lg' : 'text-slate-400 hover:text-slate-200'
            }`}
          >
            <Grid className="w-4 h-4" />
            Mappa & Posizioni
          </button>

          <button
            onClick={() => setActiveTab('history')}
            className={`flex-1 min-w-[120px] py-2.5 px-4 rounded-lg text-xs font-bold flex items-center justify-center gap-2 transition ${
              activeTab === 'history' ? 'bg-indigo-600 text-white shadow-lg' : 'text-slate-400 hover:text-slate-200'
            }`}
          >
            <History className="w-4 h-4" />
            Storico Audit
          </button>
        </div>

        {/* TAB 1: SCANNER & CARICO/SCARICO */}
        {activeTab === 'scan' && (
          <div className="grid grid-cols-1 lg:grid-cols-12 gap-6">
            
            {/* Left Column: Camera / OCR Selector */}
            <div className="lg:col-span-5 space-y-4">
              
              {/* Operation Selector */}
              <div className="bg-slate-900 p-1.5 rounded-xl border border-slate-800 flex gap-2">
                <button
                  type="button"
                  onClick={() => setOperationType('inbound')}
                  className={`flex-1 py-2 rounded-lg text-xs font-bold flex items-center justify-center gap-1.5 transition ${
                    operationType === 'inbound' 
                      ? 'bg-emerald-600 text-white shadow' 
                      : 'text-slate-400 hover:text-slate-200'
                  }`}
                >
                  <ArrowDownLeft className="w-4 h-4" /> CARICO (Stoccaggio)
                </button>
                <button
                  type="button"
                  onClick={() => setOperationType('outbound')}
                  className={`flex-1 py-2 rounded-lg text-xs font-bold flex items-center justify-center gap-1.5 transition ${
                    operationType === 'outbound' 
                      ? 'bg-amber-600 text-white shadow' 
                      : 'text-slate-400 hover:text-slate-200'
                  }`}
                >
                  <ArrowUpRight className="w-4 h-4" /> SCARICO (Prelievo)
                </button>
              </div>

              {/* Live Scanner / OCR Upload */}
              <div className="bg-slate-800 rounded-xl border border-slate-700 p-4 flex flex-col items-center">
                <div className="w-full flex items-center justify-between mb-3">
                  <span className="text-xs font-semibold text-slate-300 flex items-center gap-2">
                    <FileText className="w-4 h-4 text-indigo-400" />
                    Lettore OCR & Inquadramento Fotocamera
                  </span>
                  {isCameraActive && (
                    <span className="flex items-center gap-1 text-[10px] text-emerald-400 font-bold animate-pulse">
                      <span className="w-2 h-2 rounded-full bg-emerald-400"></span> Live
                    </span>
                  )}
                </div>

                {/* Upload Button */}
                <div className="w-full flex flex-col gap-2 mb-3">
                  <label className="w-full bg-indigo-600 hover:bg-indigo-500 text-white py-2.5 px-4 rounded-lg text-xs font-bold cursor-pointer flex items-center justify-center gap-2 shadow transition">
                    {isProcessingOcr ? (
                      <>
                        <Loader2 className="w-4 h-4 animate-spin" /> Analisi OCR in corso...
                      </>
                    ) : (
                      <>
                        <Upload className="w-4 h-4" /> Carica Foto Etichetta (OCR)
                      </>
                    )}
                    <input
                      type="file"
                      accept="image/*"
                      capture="environment"
                      onChange={handleImageOCRUpload}
                      disabled={isProcessingOcr}
                      className="hidden"
                    />
                  </label>
                </div>

                {/* Camera Feed */}
                <div className="w-full aspect-video bg-slate-950 rounded-lg border-2 border-dashed border-slate-700 relative overflow-hidden flex flex-col items-center justify-center">
                  {isCameraActive ? (
                    <div className="relative w-full h-full bg-black">
                      <video ref={videoRef} autoPlay playsInline muted className="w-full h-full object-cover" />
                      <div className="absolute inset-0 border-2 border-indigo-500/50 m-6 rounded-lg pointer-events-none flex items-center justify-center">
                        <div className="w-full h-0.5 bg-red-500/80 animate-bounce"></div>
                      </div>
                    </div>
                  ) : (
                    <div className="text-center p-4">
                      <Camera className="w-8 h-8 text-slate-600 mx-auto mb-2" />
                      <p className="text-[11px] text-slate-400 max-w-xs mb-3">
                        Inquadra l'etichetta per compilare i campi del materiale.
                      </p>
                      <button
                        type="button"
                        onClick={startCamera}
                        className="bg-slate-700 hover:bg-slate-600 text-white px-3 py-1.5 rounded text-xs font-semibold inline-flex items-center gap-1 shadow"
                      >
                        <Camera className="w-3.5 h-3.5" /> Apri Fotocamera Live
                      </button>
                    </div>
                  )}
                </div>

                {/* Test Preset Button */}
                <div className="w-full mt-4 pt-3 border-t border-slate-700">
                  <label className="text-[11px] font-semibold text-slate-300 block mb-2 flex items-center gap-1.5">
                    <FileCheck className="w-3.5 h-3.5 text-emerald-400" />
                    Test Rapido Etichetta "MiLAgro / ARPA":
                  </label>
                  <button
                    type="button"
                    onClick={loadExactLabelSample}
                    className="w-full bg-emerald-950/60 hover:bg-emerald-900/80 border border-emerald-500/60 text-emerald-200 text-xs py-2 px-3 rounded-lg flex items-center justify-between font-medium transition"
                  >
                    <span>🧪 Carica 10200001 (Potassio Cloruro)</span>
                    <span className="text-[10px] bg-emerald-800 text-white px-1.5 py-0.5 rounded">OK</span>
                  </button>
                </div>

              </div>
            </div>

            {/* Right Column: Form */}
            <div className="lg:col-span-7">
              <div className="bg-slate-800 rounded-xl border border-slate-700 p-6 shadow-xl">
                <div className="flex items-center justify-between mb-4 pb-3 border-b border-slate-700">
                  <div>
                    <h2 className="text-base font-bold text-white flex items-center gap-2">
                      {operationType === 'inbound' ? (
                        <span className="text-emerald-400 flex items-center gap-1.5">
                          <ArrowDownLeft className="w-5 h-5" /> Scheda Carico Materiale
                        </span>
                      ) : (
                        <span className="text-amber-400 flex items-center gap-1.5">
                          <ArrowUpRight className="w-5 h-5" /> Scheda Prelievo / Scarico
                        </span>
                      )}
                    </h2>
                    <p className="text-xs text-slate-400 mt-0.5">
                      Operatore attivo: <strong className="text-indigo-300">{currentUser.name}</strong>
                    </p>
                  </div>
                </div>

                {operationType === 'inbound' ? (
                  <form onSubmit={handleStockIn} className="space-y-4">
                    <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
                      <div>
                        <label className="block text-xs font-semibold text-slate-300 mb-1 flex items-center gap-1">
                          <Tag className="w-3.5 h-3.5 text-indigo-400" /> CODICE PRODOTTO *
                        </label>
                        <input
                          type="text"
                          required
                          placeholder="es. 10200001"
                          value={scannedData.productCode}
                          onChange={(e) => handleInputChange('productCode', e.target.value)}
                          className="w-full bg-slate-900 border border-slate-700 rounded-lg px-3 py-2 text-xs text-white font-mono uppercase focus:ring-2 focus:ring-indigo-500 focus:outline-none"
                        />
                      </div>

                      <div>
                        <label className="block text-xs font-semibold text-slate-300 mb-1 flex items-center gap-1">
                          <Layers className="w-3.5 h-3.5 text-indigo-400" /> LOTTO *
                        </label>
                        <input
                          type="text"
                          required
                          placeholder="es. 580ARPA"
                          value={scannedData.lot}
                          onChange={(e) => handleInputChange('lot', e.target.value)}
                          className="w-full bg-slate-900 border border-slate-700 rounded-lg px-3 py-2 text-xs text-white font-mono uppercase focus:ring-2 focus:ring-indigo-500 focus:outline-none"
                        />
                      </div>
                    </div>

                    <div>
                      <label className="block text-xs font-semibold text-slate-300 mb-1">
                        DESCRIZIONE MATERIALE
                      </label>
                      <input
                        type="text"
                        placeholder="es. POTASSIO CLORURO 60-62% K2O BB"
                        value={scannedData.description}
                        onChange={(e) => handleInputChange('description', e.target.value)}
                        className="w-full bg-slate-900 border border-slate-700 rounded-lg px-3 py-2 text-xs text-white focus:ring-2 focus:ring-indigo-500 focus:outline-none"
                      />
                    </div>

                    <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
                      <div>
                        <label className="block text-xs font-semibold text-slate-300 mb-1 flex items-center gap-1">
                          <Truck className="w-3.5 h-3.5 text-indigo-400" /> FORNITORE
                        </label>
                        <input
                          type="text"
                          placeholder="es. ARPA"
                          value={scannedData.supplier}
                          onChange={(e) => handleInputChange('supplier', e.target.value)}
                          className="w-full bg-slate-900 border border-slate-700 rounded-lg px-3 py-2 text-xs text-white focus:ring-2 focus:ring-indigo-500 focus:outline-none"
                        />
                      </div>

                      <div>
                        <label className="block text-xs font-semibold text-slate-300 mb-1 flex items-center gap-1">
                          <Calendar className="w-3.5 h-3.5 text-indigo-400" /> DATA RICEZIONE
                        </label>
                        <input
                          type="date"
                          value={scannedData.receiptDate}
                          onChange={(e) => handleInputChange('receiptDate', e.target.value)}
                          className="w-full bg-slate-900 border border-slate-700 rounded-lg px-3 py-2 text-xs text-white focus:ring-2 focus:ring-indigo-500 focus:outline-none"
                        />
                      </div>
                    </div>

                    <div className="grid grid-cols-1 sm:grid-cols-3 gap-4">
                      <div className="sm:col-span-2">
                        <label className="block text-xs font-semibold text-slate-300 mb-1">
                          QUANTITÀ *
                        </label>
                        <input
                          type="number"
                          min="1"
                          required
                          placeholder="es. 600"
                          value={scannedData.quantity}
                          onChange={(e) => handleInputChange('quantity', e.target.value)}
                          className="w-full bg-slate-900 border border-slate-700 rounded-lg px-3 py-2 text-xs text-white font-bold focus:ring-2 focus:ring-indigo-500 focus:outline-none"
                        />
                      </div>

                      <div>
                        <label className="block text-xs font-semibold text-slate-300 mb-1">
                          UNITÀ
                        </label>
                        <select
                          value={scannedData.unit}
                          onChange={(e) => handleInputChange('unit', e.target.value)}
                          className="w-full bg-slate-900 border border-slate-700 rounded-lg px-3 py-2 text-xs text-white focus:ring-2 focus:ring-indigo-500 focus:outline-none"
                        >
                          <option value="Kg">Kg</option>
                          <option value="Pz">Pz</option>
                          <option value="L">L</option>
                          <option value="Pallet">Pallet</option>
                        </select>
                      </div>
                    </div>

                    <div className="pt-3 border-t border-slate-700">
                      <label className="block text-xs font-semibold text-amber-400 mb-2 flex items-center gap-1">
                        <MapPin className="w-4 h-4" /> ASSEGNA POSIZIONE LIBERA *
                      </label>
                      <select
                        required
                        value={scannedData.selectedLocationId}
                        onChange={(e) => handleInputChange('selectedLocationId', e.target.value)}
                        className="w-full bg-slate-900 border border-indigo-500/50 rounded-lg px-3 py-2.5 text-xs text-white font-medium focus:ring-2 focus:ring-indigo-500 focus:outline-none"
                      >
                        <option value="">-- Seleziona Ubicazione --</option>
                        {locations
                          .filter(l => !l.occupied)
                          .map(loc => (
                            <option key={loc.id} value={loc.id}>
                              🟩 {loc.code}
                            </option>
                          ))}
                      </select>
                    </div>

                    <button
                      type="submit"
                      className="w-full mt-2 bg-emerald-600 hover:bg-emerald-500 text-white font-bold py-3 px-4 rounded-lg shadow-lg flex items-center justify-center gap-2 transition"
                    >
                      <Plus className="w-4 h-4" /> CONFERMA CARICO IN MAGAZZINO
                    </button>
                  </form>
                ) : (
                  <div className="space-y-3">
                    {inventory.length === 0 ? (
                      <div className="p-8 text-center text-slate-500 border border-dashed border-slate-700 rounded-lg">
                        Nessun articolo a magazzino.
                      </div>
                    ) : (
                      <div className="space-y-3 max-h-[380px] overflow-y-auto pr-1">
                        {inventory.map(item => (
                          <div 
                            key={item.id} 
                            className="bg-slate-900 p-3.5 rounded-lg border border-slate-700 flex flex-col sm:flex-row items-start sm:items-center justify-between gap-3"
                          >
                            <div className="space-y-0.5">
                              <div className="flex items-center gap-2">
                                <span className="font-bold text-amber-400 text-xs font-mono">{item.productCode}</span>
                                <span className="text-[10px] bg-slate-800 px-2 py-0.5 rounded text-slate-300 font-mono">Lotto: {item.lot}</span>
                              </div>
                              <p className="text-xs text-slate-200 font-medium">{item.description}</p>
                              <div className="text-[11px] text-slate-400 flex items-center gap-3">
                                <span>Qtà: <strong className="text-white">{item.quantity} {item.unit}</strong></span>
                                <span className="text-indigo-400 flex items-center gap-1"><MapPin className="w-3 h-3" /> {item.locationCode}</span>
                              </div>
                            </div>

                            <button
                              type="button"
                              onClick={() => handleStockOut(item)}
                              className="bg-amber-600 hover:bg-amber-500 text-white px-3 py-1.5 rounded text-xs font-bold flex items-center gap-1 transition self-end sm:self-center"
                            >
                              <ArrowUpRight className="w-3.5 h-3.5" /> Prelievo
                            </button>
                          </div>
                        ))}
                      </div>
                    )}
                  </div>
                )}
              </div>
            </div>

          </div>
        )}

        {/* TAB 2: INVENTORY TABLE */}
        {activeTab === 'inventory' && (
          <div className="bg-slate-800 rounded-xl border border-slate-700 p-6 shadow-xl">
            <div className="flex flex-col sm:flex-row sm:items-center justify-between gap-4 mb-6">
              <div>
                <h2 className="text-base font-bold text-white flex items-center gap-2">
                  <Package className="w-5 h-5 text-indigo-400" /> Registro Giacenze Magazzino
                </h2>
                <p className="text-xs text-slate-400">Prodotti stoccati in magazzino</p>
              </div>

              <div className="flex flex-col sm:flex-row items-center gap-3">
                <div className="relative w-full sm:w-64">
                  <Search className="w-3.5 h-3.5 text-slate-400 absolute left-3 top-3" />
                  <input
                    type="text"
                    placeholder="Cerca Codice, Descrizione, Lotto..."
                    value={searchTerm}
                    onChange={(e) => setSearchTerm(e.target.value)}
                    className="w-full bg-slate-900 border border-slate-700 rounded-lg pl-9 pr-3 py-2 text-xs text-white focus:outline-none focus:ring-2 focus:ring-indigo-500"
                  />
                </div>

                <select
                  value={filterLocation}
                  onChange={(e) => setFilterLocation(e.target.value)}
                  className="w-full sm:w-auto bg-slate-900 border border-slate-700 rounded-lg px-3 py-2 text-xs text-slate-300 focus:outline-none"
                >
                  <option value="ALL">Tutte le corsie</option>
                  {availableAisles.map(aisle => (
                    <option key={aisle} value={aisle.replace(/\s+/g, '')}>{aisle}</option>
                  ))}
                </select>
              </div>
            </div>

            <div className="overflow-x-auto">
              <table className="w-full text-left text-xs text-slate-300">
                <thead className="bg-slate-900/80 text-slate-400 uppercase font-semibold border-b border-slate-700">
                  <tr>
                    <th className="p-3">Codice & Descrizione</th>
                    <th className="p-3">Lotto</th>
                    <th className="p-3">Fornitore</th>
                    <th className="p-3">Giacenza</th>
                    <th className="p-3">Ubicazione</th>
                    <th className="p-3 text-right">Azione</th>
                  </tr>
                </thead>
                <tbody className="divide-y divide-slate-700/50">
                  {filteredInventory.length === 0 ? (
                    <tr>
                      <td colSpan="6" className="p-8 text-center text-slate-500">
                        Nessun articolo trovato.
                      </td>
                    </tr>
                  ) : (
                    filteredInventory.map((item) => (
                      <tr key={item.id} className="hover:bg-slate-750 transition">
                        <td className="p-3">
                          <div className="font-bold text-white font-mono">{item.productCode}</div>
                          {item.description && <div className="text-[11px] text-slate-400">{item.description}</div>}
                        </td>
                        <td className="p-3 font-mono text-indigo-300">{item.lot}</td>
                        <td className="p-3">{item.supplier}</td>
                        <td className="p-3">
                          <span className="font-bold text-emerald-400">{item.quantity}</span> {item.unit}
                        </td>
                        <td className="p-3">
                          <span className="inline-flex items-center gap-1 bg-slate-900 border border-slate-700 px-2.5 py-1 rounded text-amber-300 font-medium">
                            <MapPin className="w-3 h-3 text-amber-400" />
                            {item.locationCode}
                          </span>
                        </td>
                        <td className="p-3 text-right">
                          <button
                            onClick={() => {
                              setActiveTab('scan');
                              setOperationType('outbound');
                            }}
                            className="bg-amber-600/80 hover:bg-amber-600 text-white px-2.5 py-1 rounded text-xs font-semibold inline-flex items-center gap-1 transition"
                          >
                            <ArrowUpRight className="w-3.5 h-3.5" /> Prelievo
                          </button>
                        </td>
                      </tr>
                    ))
                  )}
                </tbody>
              </table>
            </div>
          </div>
        )}

        {/* TAB 3: LOCATION MAP & EDITING */}
        {activeTab === 'map' && (
          <div className="bg-slate-800 rounded-xl border border-slate-700 p-6 shadow-xl space-y-6">
            <div className="flex flex-col sm:flex-row sm:items-center justify-between gap-4 pb-4 border-b border-slate-700">
              <div>
                <h2 className="text-base font-bold text-white flex items-center gap-2">
                  <Grid className="w-5 h-5 text-indigo-400" /> Gestione & Personalizzazione Posizioni
                </h2>
                <p className="text-xs text-slate-400">Rinomina o aggiungi baie di magazzino</p>
              </div>

              <button
                type="button"
                onClick={() => setIsAddingNewLoc(!isAddingNewLoc)}
                className="bg-indigo-600 hover:bg-indigo-500 text-white text-xs px-3 py-2 rounded-lg font-semibold flex items-center gap-1.5 shadow"
              >
                <PlusCircle className="w-4 h-4" /> Nuova Posizione
              </button>
            </div>

            {/* Form per nuova posizione */}
            {isAddingNewLoc && (
              <form onSubmit={handleAddNewLocation} className="bg-slate-900 p-4 rounded-xl border border-indigo-500/40 space-y-3">
                <h3 className="text-xs font-bold text-indigo-400 uppercase tracking-wider flex items-center gap-1.5">
                  <PlusCircle className="w-4 h-4" /> Aggiungi Nuova Posizione
                </h3>
                <div className="grid grid-cols-1 sm:grid-cols-2 gap-3">
                  <div>
                    <label className="block text-[11px] text-slate-400 mb-1">Corsia / Area:</label>
                    <input
                      type="text"
                      placeholder="es. Corsia A o Zona Chimica"
                      value={newLocData.aisle}
                      onChange={(e) => setNewLocData({ ...newLocData, aisle: e.target.value })}
                      className="w-full bg-slate-800 border border-slate-700 rounded px-3 py-1.5 text-xs text-white"
                      required
                    />
                  </div>
                  <div>
                    <label className="block text-[11px] text-slate-400 mb-1">Denominazione Completa:</label>
                    <input
                      type="text"
                      placeholder="es. Corsia A - Scaffale 04 - Ripiano C"
                      value={newLocData.code}
                      onChange={(e) => setNewLocData({ ...newLocData, code: e.target.value })}
                      className="w-full bg-slate-800 border border-slate-700 rounded px-3 py-1.5 text-xs text-white"
                      required
                    />
                  </div>
                </div>
                <div className="flex justify-end gap-2 pt-2">
                  <button type="button" onClick={() => setIsAddingNewLoc(false)} className="px-3 py-1 text-xs text-slate-400">
                    Annulla
                  </button>
                  <button type="submit" className="bg-emerald-600 hover:bg-emerald-500 text-white text-xs px-3 py-1.5 rounded font-semibold">
                    Salva Posizione
                  </button>
                </div>
              </form>
            )}

            {/* Display Aisles */}
            {availableAisles.map(aisle => {
              const aisleLocs = locations.filter(l => l.aisle === aisle);
              return (
                <div key={aisle} className="bg-slate-900 p-4 rounded-xl border border-slate-700">
                  <h3 className="text-xs font-bold text-indigo-300 mb-3 uppercase tracking-wider flex items-center gap-2">
                    <span className="w-2 h-2 rounded-full bg-indigo-400"></span>
                    {aisle}
                  </h3>

                  <div className="grid grid-cols-1 sm:grid-cols-3 gap-3">
                    {aisleLocs.map(loc => (
                      <div
                        key={loc.id}
                        className={`p-3 rounded-lg border flex flex-col justify-between ${
                          loc.occupied
                            ? 'bg-indigo-950/60 border-indigo-500/80 text-indigo-100'
                            : 'bg-emerald-950/20 border-emerald-800/60 text-slate-400'
                        }`}
                      >
                        <div>
                          {editingLocId === loc.id ? (
                            <div className="space-y-2 mb-2">
                              <input
                                type="text"
                                value={editLocName}
                                onChange={(e) => setEditLocName(e.target.value)}
                                className="w-full bg-slate-950 border border-amber-500 rounded px-2 py-1 text-xs text-white"
                                autoFocus
                              />
                              <div className="flex items-center gap-2">
                                <button
                                  type="button"
                                  onClick={() => saveEditedLocation(loc.id)}
                                  className="bg-emerald-600 hover:bg-emerald-500 text-white px-2 py-1 rounded text-[11px] font-bold flex items-center gap-1"
                                >
                                  <Save className="w-3 h-3" /> Salva
                                </button>
                                <button type="button" onClick={() => setEditingLocId(null)} className="text-slate-400 text-[11px]">
                                  Annulla
                                </button>
                              </div>
                            </div>
                          ) : (
                            <div className="flex items-center justify-between gap-2 mb-2">
                              <span className="font-bold text-xs text-white truncate" title={loc.code}>
                                {loc.code}
                              </span>
                              <div className="flex items-center gap-1 shrink-0">
                                <button
                                  type="button"
                                  onClick={() => startEditLocation(loc)}
                                  className="p-1 text-slate-400 hover:text-indigo-300 rounded"
                                  title="Rinomina"
                                >
                                  <Edit3 className="w-3.5 h-3.5" />
                                </button>
                                {!loc.occupied && (
                                  <button
                                    type="button"
                                    onClick={() => handleDeleteLocation(loc)}
                                    className="p-1 text-slate-400 hover:text-red-400 rounded"
                                    title="Elimina"
                                  >
                                    <Trash2 className="w-3.5 h-3.5" />
                                  </button>
                                )}
                              </div>
                            </div>
                          )}

                          <span className={`text-[9px] px-2 py-0.5 rounded font-bold ${
                            loc.occupied ? 'bg-indigo-600 text-white' : 'bg-emerald-900/60 text-emerald-300'
                          }`}>
                            {loc.occupied ? 'OCCUPATO' : 'LIBERO'}
                          </span>
                        </div>

                        {loc.occupied && loc.item ? (
                          <div className="space-y-0.5 bg-slate-900/80 p-2 rounded border border-indigo-900/80 text-[11px] mt-2">
                            <p className="font-bold text-amber-300 truncate">{loc.item.productCode}</p>
                            <p className="text-slate-300">Lotto: <span className="font-mono">{loc.item.lot}</span></p>
                            <p className="text-slate-400">Qtà: <strong className="text-white">{loc.item.quantity} {loc.item.unit}</strong></p>
                          </div>
                        ) : (
                          <p className="text-[10px] text-slate-500 italic py-2 text-center">Posizione Libera</p>
                        )}
                      </div>
                    ))}
                  </div>
                </div>
              );
            })}
          </div>
        )}

        {/* TAB 4: AUDIT LOG */}
        {activeTab === 'history' && (
          <div className="bg-slate-800 rounded-xl border border-slate-700 p-6 shadow-xl">
            <div className="flex items-center justify-between mb-6 pb-3 border-b border-slate-700">
              <div>
                <h2 className="text-base font-bold text-white flex items-center gap-2">
                  <History className="w-5 h-5 text-indigo-400" /> Registro Audit Operazioni Utenti
                </h2>
                <p className="text-xs text-slate-400">Storico tracciabilità completa</p>
              </div>
            </div>

            <div className="space-y-3">
              {historyLogs.map(log => (
                <div 
                  key={log.id} 
                  className="bg-slate-900 p-3.5 rounded-lg border border-slate-700 flex flex-col sm:flex-row sm:items-center justify-between gap-3 text-xs"
                >
                  <div className="flex items-center gap-3">
                    <div className={`p-2 rounded font-bold text-[11px] flex items-center justify-center min-w-[75px] ${
                      log.type === 'CARICO' 
                        ? 'bg-emerald-900/60 text-emerald-300 border border-emerald-700' 
                        : 'bg-amber-900/60 text-amber-300 border border-amber-700'
                    }`}>
                      {log.type === 'CARICO' ? <ArrowDownLeft className="w-3.5 h-3.5 mr-1" /> : <ArrowUpRight className="w-3.5 h-3.5 mr-1" />}
                      {log.type}
                    </div>

                    <div>
                      <div className="flex items-center gap-2">
                        <span className="font-bold text-white font-mono">{log.productCode}</span>
                        {log.description && <span className="text-slate-300">({log.description})</span>}
                        <span className="text-slate-400 font-mono">(Lotto: {log.lot})</span>
                      </div>
                      <p className="text-slate-400 mt-0.5">
                        Ubicazione: <span className="text-indigo-300 font-medium">{log.location}</span>
                      </p>
                      <p className="text-[10px] text-slate-500 mt-0.5 flex items-center gap-1">
                        <User className="w-3 h-3 text-indigo-400" /> Esecutore: <strong className="text-slate-300">{log.user || 'Sistema'}</strong>
                      </p>
                    </div>
                  </div>

                  <div className="text-right sm:border-l sm:border-slate-800 sm:pl-4">
                    <p className="font-bold text-white">{log.qty}</p>
                    <p className="text-[10px] text-slate-500">{log.timestamp}</p>
                  </div>
                </div>
              ))}
            </div>
          </div>
        )}

      </main>
    </div>
  );
}
```
