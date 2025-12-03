# craft-fair-tracker
Craft fair sales &amp; profit tracking app
import React, { useState, useEffect, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { 
  getAuth, 
  onAuthStateChanged, 
  signInAnonymously, 
  signInWithCustomToken 
} from 'firebase/auth';
import { 
  getFirestore, 
  collection, 
  addDoc, 
  onSnapshot, 
  doc, 
  deleteDoc, 
  updateDoc, 
  serverTimestamp, 
  query, 
  orderBy 
} from 'firebase/firestore';
import { 
  Plus, 
  Trash2, 
  Camera, 
  DollarSign, 
  TrendingUp, 
  Package, 
  List, 
  ShoppingBag, 
  X,
  CreditCard,
  Save,
  Image as ImageIcon
} from 'lucide-react';

// --- Firebase Initialization ---
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';

// --- Components ---

// Simple Card Component
const Card = ({ children, className = "" }) => (
  <div className={`bg-white rounded-xl shadow-sm border border-slate-100 ${className}`}>
    {children}
  </div>
);

// Button Component
const Button = ({ children, onClick, variant = 'primary', className = "", disabled = false, icon: Icon }) => {
  const baseStyle = "flex items-center justify-center px-4 py-3 rounded-lg font-medium transition-all active:scale-95 touch-manipulation";
  const variants = {
    primary: "bg-indigo-600 text-white shadow-md shadow-indigo-200 hover:bg-indigo-700",
    secondary: "bg-white text-slate-700 border border-slate-200 hover:bg-slate-50",
    danger: "bg-red-50 text-red-600 border border-red-100 hover:bg-red-100",
    ghost: "bg-transparent text-slate-500 hover:bg-slate-100"
  };

  return (
    <button 
      onClick={onClick} 
      disabled={disabled}
      className={`${baseStyle} ${variants[variant]} ${disabled ? 'opacity-50 cursor-not-allowed' : ''} ${className}`}
    >
      {Icon && <Icon size={18} className="mr-2" />}
      {children}
    </button>
  );
};

// Input Component
const Input = ({ label, type = "text", value, onChange, placeholder, prefix, step }) => (
  <div className="flex flex-col gap-1 mb-3">
    {label && <label className="text-xs font-semibold text-slate-500 uppercase tracking-wide">{label}</label>}
    <div className="relative">
      {prefix && (
        <div className="absolute left-3 top-1/2 -translate-y-1/2 text-slate-400">
          {prefix}
        </div>
      )}
      <input
        type={type}
        value={value}
        onChange={onChange}
        step={step}
        placeholder={placeholder}
        className={`w-full bg-slate-50 border border-slate-200 rounded-lg py-3 px-3 text-slate-800 focus:outline-none focus:ring-2 focus:ring-indigo-500 transition-all ${prefix ? 'pl-8' : ''}`}
      />
    </div>
  </div>
);

// Image Upload & Resizer Component
const ImageUploader = ({ onImageProcessed, currentImage }) => {
  const fileInputRef = useRef(null);
  const [preview, setPreview] = useState(currentImage || null);
  const [isProcessing, setIsProcessing] = useState(false);

  const handleFileChange = (e) => {
    const file = e.target.files[0];
    if (!file) return;

    setIsProcessing(true);
    const reader = new FileReader();
    reader.onload = (event) => {
      const img = new Image();
      img.onload = () => {
        const canvas = document.createElement('canvas');
        const MAX_WIDTH = 400; // Resize to max 400px width for Firestore storage limit safety
        const scaleSize = MAX_WIDTH / img.width;
        canvas.width = MAX_WIDTH;
        canvas.height = img.height * scaleSize;

        const ctx = canvas.getContext('2d');
        ctx.drawImage(img, 0, 0, canvas.width, canvas.height);
        
        const dataUrl = canvas.toDataURL('image/jpeg', 0.7); // Compress to JPEG 70%
        setPreview(dataUrl);
        onImageProcessed(dataUrl);
        setIsProcessing(false);
      };
      img.src = event.target.result;
    };
    reader.readAsDataURL(file);
  };

  return (
    <div className="mb-4">
      <label className="text-xs font-semibold text-slate-500 uppercase tracking-wide mb-1 block">Item Photo</label>
      <div 
        onClick={() => fileInputRef.current?.click()}
        className="relative w-full h-40 bg-slate-100 border-2 border-dashed border-slate-300 rounded-lg flex flex-col items-center justify-center cursor-pointer overflow-hidden hover:bg-slate-200 transition-colors"
      >
        {preview ? (
          <img src={preview} alt="Preview" className="w-full h-full object-cover" />
        ) : (
          <div className="flex flex-col items-center text-slate-400">
            <Camera size={32} className="mb-2" />
            <span className="text-sm">Tap to add photo</span>
          </div>
        )}
        {isProcessing && (
          <div className="absolute inset-0 bg-black/50 flex items-center justify-center text-white text-sm font-medium">
            Processing...
          </div>
        )}
      </div>
      <input 
        type="file" 
        ref={fileInputRef} 
        onChange={handleFileChange} 
        accept="image/*" 
        className="hidden" 
      />
      {preview && (
        <button 
          onClick={(e) => { e.stopPropagation(); setPreview(null); onImageProcessed(null); }}
          className="mt-2 text-xs text-red-500 font-medium flex items-center"
        >
          <Trash2 size={12} className="mr-1" /> Remove Photo
        </button>
      )}
    </div>
  );
};

// --- Main Application ---
export default function CraftFairTracker() {
  const [user, setUser] = useState(null);
  const [activeTab, setActiveTab] = useState('pos'); // pos, inventory, transactions, stats
  
  // Data States
  const [inventory, setInventory] = useState([]);
  const [transactions, setTransactions] = useState([]);
  
  // UI States for Forms
  const [isAddingItem, setIsAddingItem] = useState(false);
  const [cart, setCart] = useState([]);
  
  // New Item Form State
  const [newItem, setNewItem] = useState({
    name: '',
    price: '',
    cost: '',
    image: null
  });

  // Authentication
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

  // Data Sync
  useEffect(() => {
    if (!user) return;

    // Listen to Inventory
    const inventoryRef = collection(db, 'artifacts', appId, 'users', user.uid, 'inventory');
    const unsubInv = onSnapshot(inventoryRef, (snapshot) => {
      const items = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      setInventory(items.sort((a, b) => a.name.localeCompare(b.name)));
    }, (err) => console.error("Inv Error:", err));

    // Listen to Transactions
    const txRef = collection(db, 'artifacts', appId, 'users', user.uid, 'sales');
    // Note: In a real app we would use orderBy('date', 'desc') but adhering to RULE 2 (No Complex Queries initially)
    // we fetch all and sort in memory.
    const unsubTx = onSnapshot(txRef, (snapshot) => {
      const txs = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      // Sort by date desc
      txs.sort((a, b) => {
        const dateA = a.date?.toDate ? a.date.toDate() : new Date(0);
        const dateB = b.date?.toDate ? b.date.toDate() : new Date(0);
        return dateB - dateA;
      });
      setTransactions(txs);
    }, (err) => console.error("Tx Error:", err));

    return () => {
      unsubInv();
      unsubTx();
    };
  }, [user]);

  // --- Handlers ---

  const handleAddItem = async () => {
    if (!newItem.name || !newItem.price) return;
    
    try {
      await addDoc(collection(db, 'artifacts', appId, 'users', user.uid, 'inventory'), {
        name: newItem.name,
        price: parseFloat(newItem.price),
        cost: parseFloat(newItem.cost) || 0,
        image: newItem.image,
        createdAt: serverTimestamp()
      });
      setNewItem({ name: '', price: '', cost: '', image: null });
      setIsAddingItem(false);
    } catch (e) {
      console.error("Error adding item: ", e);
    }
  };

  const deleteInventoryItem = async (id) => {
    if (confirm('Delete this item from inventory?')) {
      await deleteDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'inventory', id));
    }
  };

  const addToCart = (item) => {
    setCart(prev => {
      const existing = prev.find(i => i.id === item.id);
      if (existing) {
        return prev.map(i => i.id === item.id ? { ...i, qty: i.qty + 1 } : i);
      }
      return [...prev, { ...item, qty: 1 }];
    });
  };

  const removeFromCart = (itemId) => {
    setCart(prev => prev.filter(i => i.id !== itemId));
  };

  const updateCartQty = (itemId, change) => {
    setCart(prev => prev.map(i => {
      if (i.id === itemId) {
        const newQty = Math.max(1, i.qty + change);
        return { ...i, qty: newQty };
      }
      return i;
    }));
  };

  const processSale = async (paymentMethod) => {
    if (cart.length === 0) return;

    const totalSale = cart.reduce((acc, item) => acc + (item.price * item.qty), 0);
    const totalCost = cart.reduce((acc, item) => acc + (item.cost * item.qty), 0);
    const profit = totalSale - totalCost;

    const saleData = {
      date: serverTimestamp(),
      items: cart,
      total: totalSale,
      cost: totalCost,
      profit: profit,
      paymentMethod: paymentMethod,
      itemCount: cart.reduce((acc, item) => acc + item.qty, 0)
    };

    try {
      await addDoc(collection(db, 'artifacts', appId, 'users', user.uid, 'sales'), saleData);
      setCart([]);
      setActiveTab('transactions'); // Feedback by switching tabs briefly or showing success? Let's stay on POS but clear cart
    } catch (e) {
      console.error("Sale Error:", e);
    }
  };

  // --- Views ---

  const renderPOS = () => {
    const cartTotal = cart.reduce((acc, item) => acc + (item.price * item.qty), 0);

    return (
      <div className="flex flex-col h-full overflow-hidden">
        {/* Cart Preview (Collapsible or always visible? Let's make it sticky top) */}
        {cart.length > 0 && (
          <div className="bg-indigo-600 text-white p-4 shadow-md z-10">
            <div className="flex justify-between items-end mb-2">
              <span className="font-bold text-lg">Current Sale</span>
              <span className="font-bold text-2xl">${cartTotal.toFixed(2)}</span>
            </div>
            <div className="flex gap-2 mt-3 overflow-x-auto pb-1">
              {['Cash', 'Card', 'Venmo'].map(method => (
                <button
                  key={method}
                  onClick={() => processSale(method)}
                  className="flex-1 bg-white/20 hover:bg-white/30 backdrop-blur-sm py-2 px-3 rounded-lg font-medium text-sm transition-colors whitespace-nowrap"
                >
                  Pay {method}
                </button>
              ))}
            </div>
            <div className="mt-2 text-center text-xs text-indigo-200">
              Tap a payment method to complete
            </div>
          </div>
        )}

        {/* Inventory Grid for POS */}
        <div className="flex-1 overflow-y-auto p-4 bg-slate-50">
           {cart.length > 0 && (
             <div className="mb-6 bg-white rounded-xl p-3 shadow-sm border border-slate-200">
                <h3 className="text-xs font-bold text-slate-400 uppercase mb-2">Cart Items</h3>
                {cart.map(item => (
                  <div key={item.id} className="flex justify-between items-center py-2 border-b border-slate-100 last:border-0">
                    <div className="flex items-center">
                       <span className="font-medium text-slate-800">{item.name}</span>
                       <span className="text-xs text-slate-500 ml-2">${item.price} x {item.qty}</span>
                    </div>
                    <div className="flex items-center gap-2">
                      <button onClick={() => updateCartQty(item.id, -1)} className="p-1 text-slate-400 hover:text-indigo-600"><TrendingUp size={14} className="rotate-180"/></button>
                      <button onClick={() => updateCartQty(item.id, 1)} className="p-1 text-slate-400 hover:text-indigo-600"><Plus size={14} /></button>
                      <button onClick={() => removeFromCart(item.id)} className="p-1 text-red-400"><X size={14} /></button>
                    </div>
                  </div>
                ))}
             </div>
           )}

           <h3 className="text-xs font-bold text-slate-400 uppercase mb-3">Tap to Add</h3>
           {inventory.length === 0 ? (
             <div className="text-center py-12 px-4 text-slate-400">
               <Package size={48} className="mx-auto mb-3 opacity-20" />
               <p>No items yet. Go to Inventory to add products.</p>
               <Button variant="secondary" className="mt-4" onClick={() => setActiveTab('inventory')}>Go to Inventory</Button>
             </div>
           ) : (
             <div className="grid grid-cols-2 gap-3">
              {inventory.map(item => (
                <button 
                  key={item.id}
                  onClick={() => addToCart(item)}
                  className="bg-white p-3 rounded-xl shadow-sm border border-slate-100 flex flex-col items-center text-center active:scale-95 transition-transform"
                >
                  <div className="w-full aspect-square bg-slate-100 rounded-lg mb-2 overflow-hidden">
                    {item.image ? (
                      <img src={item.image} alt={item.name} className="w-full h-full object-cover" />
                    ) : (
                      <div className="w-full h-full flex items-center justify-center text-slate-300">
                        <ShoppingBag size={24} />
                      </div>
                    )}
                  </div>
                  <span className="font-medium text-slate-700 text-sm truncate w-full">{item.name}</span>
                  <span className="text-indigo-600 font-bold">${item.price.toFixed(2)}</span>
                </button>
              ))}
            </div>
           )}
        </div>
      </div>
    );
  };

  const renderInventory = () => (
    <div className="p-4 bg-slate-50 min-h-full pb-20">
      <div className="flex justify-between items-center mb-6">
        <h2 className="text-2xl font-bold text-slate-800">Inventory</h2>
        <Button onClick={() => setIsAddingItem(!isAddingItem)} icon={isAddingItem ? X : Plus}>
          {isAddingItem ? 'Cancel' : 'Add Item'}
        </Button>
      </div>

      {isAddingItem && (
        <Card className="p-4 mb-6 animate-fade-in-down">
          <h3 className="font-bold text-slate-700 mb-4">New Product</h3>
          <ImageUploader 
            currentImage={newItem.image} 
            onImageProcessed={(data) => setNewItem({...newItem, image: data})} 
          />
          <Input 
            label="Item Name" 
            placeholder="e.g. Handmade Mug" 
            value={newItem.name} 
            onChange={e => setNewItem({...newItem, name: e.target.value})} 
          />
          <div className="flex gap-3">
             <div className="flex-1">
               <Input 
                  label="Sale Price ($)" 
                  type="number" 
                  step="0.01"
                  placeholder="0.00" 
                  value={newItem.price} 
                  onChange={e => setNewItem({...newItem, price: e.target.value})} 
               />
             </div>
             <div className="flex-1">
               <Input 
                  label="Mat. Cost ($)" 
                  type="number" 
                  step="0.01"
                  placeholder="0.00" 
                  value={newItem.cost} 
                  onChange={e => setNewItem({...newItem, cost: e.target.value})} 
               />
             </div>
          </div>
          <Button className="w-full mt-2" onClick={handleAddItem} disabled={!newItem.name || !newItem.price} icon={Save}>
            Save Item
          </Button>
        </Card>
      )}

      <div className="space-y-3">
        {inventory.map(item => (
          <Card key={item.id} className="p-3 flex items-center gap-3">
            <div className="w-16 h-16 bg-slate-100 rounded-lg flex-shrink-0 overflow-hidden">
               {item.image ? (
                 <img src={item.image} alt={item.name} className="w-full h-full object-cover" />
               ) : (
                 <div className="w-full h-full flex items-center justify-center text-slate-300">
                   <ImageIcon size={20} />
                 </div>
               )}
            </div>
            <div className="flex-1 min-w-0">
              <h4 className="font-bold text-slate-800 truncate">{item.name}</h4>
              <div className="flex gap-3 text-sm">
                <span className="text-slate-500">Price: <span className="text-slate-800 font-semibold">${item.price.toFixed(2)}</span></span>
                <span className="text-slate-400">Cost: ${item.cost?.toFixed(2) || '0.00'}</span>
              </div>
            </div>
            <button 
              onClick={() => deleteInventoryItem(item.id)} 
              className="p-2 text-slate-300 hover:text-red-500 transition-colors"
            >
              <Trash2 size={18} />
            </button>
          </Card>
        ))}
      </div>
    </div>
  );

  const renderTransactions = () => (
    <div className="p-4 bg-slate-50 min-h-full pb-20">
      <h2 className="text-2xl font-bold text-slate-800 mb-6">Sales History</h2>
      {transactions.length === 0 ? (
        <div className="text-center text-slate-400 py-10">No sales yet.</div>
      ) : (
        <div className="space-y-3">
          {transactions.map(tx => (
            <Card key={tx.id} className="p-4">
              <div className="flex justify-between items-start mb-2">
                <div>
                   <span className="text-xs font-bold text-indigo-600 bg-indigo-50 px-2 py-1 rounded-full uppercase tracking-wide">
                     {tx.paymentMethod || 'Cash'}
                   </span>
                   <div className="text-xs text-slate-400 mt-2">
                     {tx.date?.toDate ? tx.date.toDate().toLocaleTimeString([], {hour: '2-digit', minute:'2-digit'}) : 'Just now'}
                   </div>
                </div>
                <div className="text-right">
                   <div className="font-bold text-xl text-slate-800">${tx.total.toFixed(2)}</div>
                   <div className="text-xs text-green-600 font-medium">Profit: +${(tx.profit || 0).toFixed(2)}</div>
                </div>
              </div>
              <div className="border-t border-slate-100 mt-3 pt-3">
                {tx.items.map((item, idx) => (
                  <div key={idx} className="flex justify-between text-sm text-slate-600 mb-1">
                    <span>{item.qty}x {item.name}</span>
                    <span>${(item.price * item.qty).toFixed(2)}</span>
                  </div>
                ))}
              </div>
            </Card>
          ))}
        </div>
      )}
    </div>
  );

  const renderStats = () => {
    const totalSales = transactions.reduce((acc, tx) => acc + tx.total, 0);
    const totalProfit = transactions.reduce((acc, tx) => acc + (tx.profit || 0), 0);
    const totalItems = transactions.reduce((acc, tx) => acc + (tx.itemCount || 0), 0);
    
    // Group by payment method
    const byMethod = transactions.reduce((acc, tx) => {
      const method = tx.paymentMethod || 'Unknown';
      acc[method] = (acc[method] || 0) + tx.total;
      return acc;
    }, {});

    return (
      <div className="p-4 bg-slate-50 min-h-full pb-20">
        <h2 className="text-2xl font-bold text-slate-800 mb-6">Performance</h2>
        
        <div className="grid grid-cols-2 gap-3 mb-6">
           <Card className="p-4 bg-indigo-600 text-white border-none">
              <div className="text-indigo-200 text-xs font-semibold uppercase mb-1">Total Sales</div>
              <div className="text-3xl font-bold">${totalSales.toFixed(2)}</div>
           </Card>
           <Card className="p-4 bg-emerald-500 text-white border-none">
              <div className="text-emerald-100 text-xs font-semibold uppercase mb-1">Net Profit</div>
              <div className="text-3xl font-bold">${totalProfit.toFixed(2)}</div>
           </Card>
        </div>

        <Card className="p-5 mb-6">
          <h3 className="text-slate-800 font-bold mb-4 flex items-center">
            <CreditCard size={18} className="mr-2 text-slate-400"/>
            Revenue by Payment Type
          </h3>
          <div className="space-y-3">
             {Object.entries(byMethod).map(([method, amount]) => (
               <div key={method} className="flex items-center justify-between">
                 <div className="flex items-center">
                    <div className={`w-2 h-2 rounded-full mr-3 ${method === 'Cash' ? 'bg-green-500' : 'bg-blue-500'}`}></div>
                    <span className="text-slate-600 font-medium">{method}</span>
                 </div>
                 <span className="text-slate-800 font-bold">${amount.toFixed(2)}</span>
               </div>
             ))}
             {Object.keys(byMethod).length === 0 && <div className="text-slate-400 text-sm">No data yet</div>}
          </div>
        </Card>

        <Card className="p-5">
           <h3 className="text-slate-800 font-bold mb-4 flex items-center">
            <Package size={18} className="mr-2 text-slate-400"/>
            Volume
          </h3>
           <div className="flex justify-between items-center py-2 border-b border-slate-50">
             <span className="text-slate-500">Items Sold</span>
             <span className="text-slate-800 font-bold text-lg">{totalItems}</span>
           </div>
           <div className="flex justify-between items-center py-2">
             <span className="text-slate-500">Avg. Transaction</span>
             <span className="text-slate-800 font-bold text-lg">
               ${transactions.length > 0 ? (totalSales / transactions.length).toFixed(2) : '0.00'}
             </span>
           </div>
        </Card>
      </div>
    );
  };

  if (!user) return (
    <div className="flex h-screen items-center justify-center bg-slate-50 text-slate-400">
      <div className="animate-pulse flex flex-col items-center">
        <Package size={48} className="mb-4" />
        <p>Loading Tracker...</p>
      </div>
    </div>
  );

  return (
    <div className="flex flex-col h-screen bg-slate-100 font-sans max-w-md mx-auto shadow-2xl relative overflow-hidden">
      {/* Top Bar */}
      <div className="bg-white border-b border-slate-200 px-4 py-3 flex justify-between items-center z-10">
        <h1 className="text-lg font-extrabold text-slate-800 tracking-tight flex items-center">
           <span className="bg-indigo-600 text-white p-1 rounded mr-2">
             <DollarSign size={16} />
           </span>
           FairTracker
        </h1>
        <div className="text-xs text-slate-400 font-medium bg-slate-50 px-2 py-1 rounded">
          {inventory.length} Items
        </div>
      </div>

      {/* Main Content Area */}
      <div className="flex-1 overflow-hidden relative">
        {activeTab === 'pos' && renderPOS()}
        {activeTab === 'inventory' && renderInventory()}
        {activeTab === 'transactions' && renderTransactions()}
        {activeTab === 'stats' && renderStats()}
      </div>

      {/* Bottom Navigation */}
      <div className="bg-white border-t border-slate-200 px-2 py-1 flex justify-around items-center h-[60px] z-20 shrink-0">
        <button 
          onClick={() => setActiveTab('pos')}
          className={`flex flex-col items-center justify-center w-full h-full space-y-1 ${activeTab === 'pos' ? 'text-indigo-600' : 'text-slate-400 hover:text-slate-600'}`}
        >
          <ShoppingBag size={20} className={activeTab === 'pos' ? 'fill-current opacity-20' : ''} />
          <span className="text-[10px] font-bold">Sell</span>
        </button>
        <button 
          onClick={() => setActiveTab('inventory')}
          className={`flex flex-col items-center justify-center w-full h-full space-y-1 ${activeTab === 'inventory' ? 'text-indigo-600' : 'text-slate-400 hover:text-slate-600'}`}
        >
          <List size={20} className={activeTab === 'inventory' ? 'fill-current opacity-20' : ''} />
          <span className="text-[10px] font-bold">Inventory</span>
        </button>
        <button 
          onClick={() => setActiveTab('transactions')}
          className={`flex flex-col items-center justify-center w-full h-full space-y-1 ${activeTab === 'transactions' ? 'text-indigo-600' : 'text-slate-400 hover:text-slate-600'}`}
        >
          <List size={20} className={activeTab === 'transactions' ? 'fill-current opacity-20' : ''} />
          <span className="text-[10px] font-bold">History</span>
        </button>
        <button 
          onClick={() => setActiveTab('stats')}
          className={`flex flex-col items-center justify-center w-full h-full space-y-1 ${activeTab === 'stats' ? 'text-indigo-600' : 'text-slate-400 hover:text-slate-600'}`}
        >
          <TrendingUp size={20} className={activeTab === 'stats' ? 'fill-current opacity-20' : ''} />
          <span className="text-[10px] font-bold">Stats</span>
        </button>
      </div>
    </div>
  );
}

