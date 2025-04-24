// Loja Fácil - Sistema Completo
import React, { useState, useEffect } from 'react';
import { initializeApp } from 'firebase/app';
import {
  getAuth,
  onAuthStateChanged,
  signInWithEmailAndPassword,
  createUserWithEmailAndPassword,
  signOut
} from 'firebase/auth';
import {
  getFirestore,
  collection,
  addDoc,
  getDocs,
  updateDoc,
  doc,
  serverTimestamp
} from 'firebase/firestore';
import logo from './path/to/your/logo.png'; // Aqui você coloca o caminho para a logo que você carregou

const firebaseConfig = {
  apiKey: "SUA_API_KEY",
  authDomain: "SEU_DOMINIO.firebaseapp.com",
  projectId: "SEU_PROJECT_ID",
  storageBucket: "SEU_BUCKET.appspot.com",
  messagingSenderId: "SENDER_ID",
  appId: "APP_ID"
};

const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);

export default function App() {
  const [user, setUser] = useState(null);
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [isLogin, setIsLogin] = useState(true);
  const [products, setProducts] = useState([]);
  const [sales, setSales] = useState([]);
  const [newProduct, setNewProduct] = useState({ nome: '', codigo: '', custo: '', preco: '', estoque: '' });
  const [sale, setSale] = useState({ produtoId: '', quantidade: '' });
  const [isPro, setIsPro] = useState(false); // Adicionando o controle de plano PRO

  useEffect(() => {
    const unsubscribe = onAuthStateChanged(auth, (currentUser) => {
      setUser(currentUser);
      if (currentUser) {
        fetchData();
      }
    });
    return unsubscribe;
  }, []);

  const handleAuth = async () => {
    try {
      if (isLogin) {
        await signInWithEmailAndPassword(auth, email, password);
      } else {
        await createUserWithEmailAndPassword(auth, email, password);
      }
    } catch (error) {
      alert(error.message);
    }
  };

  const handleLogout = () => {
    signOut(auth);
  };

  const fetchData = async () => {
    const productsSnapshot = await getDocs(collection(db, 'produtos'));
    const salesSnapshot = await getDocs(collection(db, 'vendas'));
    setProducts(productsSnapshot.docs.map(doc => ({ id: doc.id, ...doc.data() })));
    setSales(salesSnapshot.docs.map(doc => doc.data()));
  };

  const cadastrarProduto = async () => {
    if (products.length >= 80 && !isPro) return alert('Limite de 80 produtos atingido. Faça upgrade para liberar produtos ilimitados.');
    const data = { ...newProduct, custo: Number(newProduct.custo), preco: Number(newProduct.preco), estoque: Number(newProduct.estoque) };
    await addDoc(collection(db, 'produtos'), data);
    setNewProduct({ nome: '', codigo: '', custo: '', preco: '', estoque: '' });
    fetchData();
  };

  const registrarVenda = async () => {
    const produto = products.find(p => p.id === sale.produtoId);
    if (!produto || produto.estoque < Number(sale.quantidade)) return alert('Estoque insuficiente.');
    const lucro = (produto.preco - produto.custo) * Number(sale.quantidade);
    await addDoc(collection(db, 'vendas'), {
      produto: produto.nome,
      quantidade: Number(sale.quantidade),
      data: serverTimestamp(),
      lucro
    });
    await updateDoc(doc(db, 'produtos', sale.produtoId), {
      estoque: produto.estoque - Number(sale.quantidade)
    });
    setSale({ produtoId: '', quantidade: '' });
    fetchData();
  };

  const totalVendas = sales.reduce((acc, s) => acc + (s.quantidade || 0), 0);
  const totalLucro = sales.reduce((acc, s) => acc + (s.lucro || 0), 0);

  const upgradeToPro = () => {
    setIsPro(true);
    alert("Você agora tem acesso ao plano PRO, com produtos ilimitados e mais funcionalidades!");
  };

  if (!user) {
    return (
      <div className="min-h-screen flex flex-col items-center justify-center bg-gray-100 dark:bg-gray-900 text-gray-900 dark:text-white">
        <div className="w-full max-w-xs text-center">
          <img src={logo} alt="Logo Loja Fácil" className="w-32 mb-6 mx-auto" />
          <h1 className="text-xl mb-4">{isLogin ? 'Login' : 'Cadastro'}</h1>
          <input className="w-full mb-2 px-3 py-2 border rounded" type="email" placeholder="Email" value={email} onChange={(e) => setEmail(e.target.value)} />
          <input className="w-full mb-2 px-3 py-2 border rounded" type="password" placeholder="Senha" value={password} onChange={(e) => setPassword(e.target.value)} />
          <button className="w-full px-4 py-2 bg-blue-600 text-white rounded" onClick={handleAuth}>{isLogin ? 'Entrar' : 'Cadastrar'}</button>
          <p className="mt-4 text-sm text-center cursor-pointer" onClick={() => setIsLogin(!isLogin)}>{isLogin ? 'Criar uma conta' : 'Já tem uma conta? Entrar'}</p>
        </div>
      </div>
    );
  }

  return (
    <div className="min-h-screen p-4 bg-gray-100 dark:bg-gray-900 text-gray-900 dark:text-white">
      <div className="flex justify-between items-center mb-4">
        <h1 className="text-2xl font-bold">Loja Fácil - Painel</h1>
        <button className="px-4 py-2 bg-red-500 text-white rounded" onClick={handleLogout}>Sair</button>
      </div>

      {isPro ? (
        <div className="bg-green-600 text-white text-center p-2 mb-4">
          Você está no plano PRO! Produtos ilimitados disponíveis.
        </div>
      ) : (
        <div className="bg-yellow-500 text-white text-center p-2 mb-4">
          Você está no plano gratuito. Faça upgrade para liberar produtos ilimitados!
          <button className="ml-2 px-4 py-2 bg-blue-600 text-white rounded" onClick={upgradeToPro}>Upgrade para PRO</button>
        </div>
      )}

      <div className="grid grid-cols-1 md:grid-cols-2 gap-4 mb-6">
        <div className="p-4 border rounded bg-white dark:bg-gray-800">
          <h2 className="font-bold mb-2">Cadastrar Produto</h2>
          <input className="w-full mb-1 px-2 py-1 border rounded" placeholder="Nome" value={newProduct.nome} onChange={e => setNewProduct({ ...newProduct, nome: e.target.value })} />
          <input className="w-full mb-1 px-2 py-1 border rounded" placeholder="Código de Barras" value={newProduct.codigo} onChange={e => setNewProduct({ ...newProduct, codigo: e.target.value })} />
          <input className="w-full mb-1 px-2 py-1 border rounded" placeholder="Preço de Custo" value={newProduct.custo} onChange={e => setNewProduct({ ...newProduct, custo: e.target.value })} />
          <input className="w-full mb-1 px-2 py-1 border rounded" placeholder="Preço de Venda" value={newProduct.preco} onChange={e => setNewProduct({ ...newProduct, preco: e.target.value })} />
          <input className="w-full mb-2 px-2 py-1 border rounded" placeholder="Estoque" value={newProduct.estoque} onChange={e => setNewProduct({ ...newProduct, estoque: e.target.value })} />
          <button className="w-full px-4 py-2 bg-green-600 text-white rounded" onClick={cadastrarProduto}>Salvar Produto</button>
        </div>

        <div className="p-4 border rounded bg-white dark:bg-gray-800">
          <h2 className="font-bold mb-2">Registrar Venda</h2>
          <select className="w-full mb-2 px-2 py-1 border rounded" value={sale.produtoId} onChange={e => setSale({ ...sale, produtoId: e.target.value })}>
            <option value="">Selecione o produto</option>
            {products.map(p => <option key={p.id} value={p.id}>{p.nome}</option>)}
          </select>
          <input className="w-full mb-2 px-2 py-1 border rounded" placeholder="Quantidade" value={sale.quantidade} onChange={e => setSale({ ...sale, quantidade: e.target.value })} />
