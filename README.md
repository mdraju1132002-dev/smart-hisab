
import React, { useState, useEffect, useMemo } from 'react';
import { Transaction, TransactionType, FinancialSummary } from './types';
import TransactionForm from './components/TransactionForm';
import SummaryCards from './components/SummaryCards';
import InsightsPanel from './components/InsightsPanel';
import { getUSDTtoBDTRate, RateSource } from './services/geminiService';
import { BarChart, Bar, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer, Cell } from 'recharts';

const App: React.FC = () => {
  const [transactions, setTransactions] = useState<Transaction[]>(() => {
    const saved = localStorage.getItem('hisab_transactions');
    return saved ? JSON.parse(saved) : [];
  });

  const [exchangeRate, setExchangeRate] = useState<number>(() => {
    const saved = localStorage.getItem('usdt_bdt_rate');
    return saved ? parseFloat(saved) : 120.0;
  });

  const [isUpdatingRate, setIsUpdatingRate] = useState(false);
  const [rateSources, setRateSources] = useState<RateSource[]>([]);

  useEffect(() => {
    localStorage.setItem('hisab_transactions', JSON.stringify(transactions));
  }, [transactions]);

  useEffect(() => {
    localStorage.setItem('usdt_bdt_rate', exchangeRate.toString());
  }, [exchangeRate]);

  const updateRate = async () => {
    setIsUpdatingRate(true);
    const { rate, sources } = await getUSDTtoBDTRate();
    if (rate) {
      setExchangeRate(rate);
      setRateSources(sources);
    }
    setIsUpdatingRate(false);
  };

  const addTransaction = (data: Omit<Transaction, 'id'>) => {
    const newTx: Transaction = {
      ...data,
      id: crypto.randomUUID(),
    };
    setTransactions((prev) => [newTx, ...prev]);
  };

  const deleteTransaction = (id: string) => {
    setTransactions((prev) => prev.filter(t => t.id !== id));
  };

  const summary = useMemo<FinancialSummary>(() => {
    return transactions.reduce((acc, t) => {
      if (t.type === TransactionType.INCOME) {
        acc.totalIncome += t.amount;
        acc.totalBalance += t.amount;
      } else {
        acc.totalExpense += t.amount;
        acc.totalBalance -= t.amount;
      }
      return acc;
    }, { totalBalance: 0, totalIncome: 0, totalExpense: 0 });
  }, [transactions]);

  const chartData = useMemo(() => {
    return transactions.slice(0, 7).reverse().map(t => ({
      name: t.description.length > 10 ? t.description.substring(0, 8) + '...' : t.description,
      amount: t.amount,
      type: t.type
    }));
  }, [transactions]);

  const toBDT = (usdt: number) => (usdt * exchangeRate).toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 });

  return (
    <div className="min-h-screen pb-20 px-4 md:px-8 max-w-7xl mx-auto">
      <header className="py-8 flex flex-col md:flex-row md:items-center justify-between gap-4">
        <div>
          <h1 className="text-3xl font-extrabold text-slate-900 tracking-tight">Smart Hisab</h1>
          <p className="text-slate-500 font-medium">USDT to BDT Tracking & Analysis</p>
        </div>
        
        <div className="flex flex-col items-end gap-2">
          <div className="bg-white px-4 py-2 rounded-xl border border-slate-200 shadow-sm flex items-center gap-3">
            <div className="text-right">
              <p className="text-[10px] text-slate-400 uppercase font-bold leading-none">Current Rate</p>
              <p className="text-sm font-bold text-slate-700">1 USDT = ৳ {exchangeRate}</p>
            </div>
            <button 
              onClick={updateRate}
              disabled={isUpdatingRate}
              className="p-2 hover:bg-slate-50 rounded-lg text-indigo-600 transition-all disabled:opacity-50"
              title="Update Rate via AI"
            >
              <svg className={`w-5 h-5 ${isUpdatingRate ? 'animate-spin' : ''}`} fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15" />
              </svg>
            </button>
          </div>
          
          {rateSources.length > 0 && (
            <div className="flex flex-wrap justify-end gap-2 max-w-xs">
              {rateSources.map((source, idx) => (
                <a 
                  key={idx} 
                  href={source.uri} 
                  target="_blank" 
                  rel="noopener noreferrer" 
                  className="text-[10px] bg-slate-100 hover:bg-indigo-50 text-slate-600 hover:text-indigo-600 px-2 py-0.5 rounded border border-slate-200 transition-colors inline-block truncate max-w-[120px]"
                  title={source.title}
                >
                  {source.title}
                </a>
              ))}
            </div>
          )}
        </div>
      </header>

      <SummaryCards summary={summary} exchangeRate={exchangeRate} />

      <div className="grid grid-cols-1 lg:grid-cols-12 gap-8">
        <div className="lg:col-span-4 space-y-8">
          <TransactionForm onAdd={addTransaction} />
          
          <div className="bg-white p-6 rounded-2xl shadow-sm border border-slate-100">
             <div className="flex items-center justify-between mb-4">
               <h3 className="text-sm font-bold text-slate-400 uppercase">Quick Conversion</h3>
               <span className="text-[10px] font-bold text-slate-300">BASED ON ৳{exchangeRate}</span>
             </div>
             <div className="space-y-3">
                <div className="flex justify-between items-center py-2 border-b border-slate-50">
                   <span className="text-slate-600">10 USDT</span>
                   <span className="font-bold text-slate-800">৳ {toBDT(10)}</span>
                </div>
                <div className="flex justify-between items-center py-2 border-b border-slate-50">
                   <span className="text-slate-600">50 USDT</span>
                   <span className="font-bold text-slate-800">৳ {toBDT(50)}</span>
                </div>
                <div className="flex justify-between items-center py-2">
                   <span className="text-slate-600">100 USDT</span>
                   <span className="font-bold text-slate-800">৳ {toBDT(100)}</span>
                </div>
             </div>
          </div>

          <InsightsPanel transactions={transactions} />
        </div>

        <div className="lg:col-span-8 space-y-8">
          <div className="bg-white p-6 rounded-2xl shadow-sm border border-slate-100">
            <h2 className="text-xl font-bold text-slate-800 mb-6">Activity Trends (USDT)</h2>
            <div className="h-[300px] w-full">
              <ResponsiveContainer width="100%" height="100%">
                <BarChart data={chartData}>
                  <CartesianGrid strokeDasharray="3 3" vertical={false} stroke="#f1f5f9" />
                  <XAxis dataKey="name" stroke="#94a3b8" fontSize={12} tickLine={false} axisLine={false} />
                  <YAxis stroke="#94a3b8" fontSize={12} tickLine={false} axisLine={false} />
                  <Tooltip 
                    cursor={{fill: '#f8fafc'}}
                    contentStyle={{ borderRadius: '12px', border: 'none', boxShadow: '0 10px 15px -3px rgba(0,0,0,0.1)' }}
                    formatter={(value: any) => [`${value} USDT`, 'Amount']}
                  />
                  <Bar dataKey="amount" radius={[6, 6, 0, 0]}>
                    {chartData.map((entry, index) => (
                      <Cell key={`cell-${index}`} fill={entry.type === TransactionType.INCOME ? '#10b981' : '#f43f5e'} />
                    ))}
                  </Bar>
                </BarChart>
              </ResponsiveContainer>
            </div>
          </div>

          <div className="bg-white rounded-2xl shadow-sm border border-slate-100 overflow-hidden">
            <div className="p-6 border-b border-slate-100 flex items-center justify-between">
              <h2 className="text-xl font-bold text-slate-800">History (Converted to BDT)</h2>
              <span className="text-xs font-bold text-indigo-600 bg-indigo-50 px-2 py-1 rounded">
                Rate: ৳{exchangeRate}
              </span>
            </div>
            <div className="overflow-x-auto">
              <table className="w-full text-left">
                <thead className="bg-slate-50 text-slate-500 text-xs uppercase font-semibold">
                  <tr>
                    <th className="px-6 py-4">Date</th>
                    <th className="px-6 py-4">Description</th>
                    <th className="px-6 py-4">USDT</th>
                    <th className="px-6 py-4">BDT Equivalent</th>
                    <th className="px-6 py-4 text-right">Actions</th>
                  </tr>
                </thead>
                <tbody className="divide-y divide-slate-50">
                  {transactions.length === 0 ? (
                    <tr>
                      <td colSpan={5} className="px-6 py-12 text-center text-slate-400 italic">
                        No transactions recorded.
                      </td>
                    </tr>
                  ) : (
                    transactions.map((tx) => (
                      <tr key={tx.id} className="hover:bg-slate-50 transition-colors group">
                        <td className="px-6 py-4 text-sm text-slate-500">{tx.date}</td>
                        <td className="px-6 py-4">
                           <div className="font-medium text-slate-800">{tx.description}</div>
                           <div className="text-xs text-slate-400">{tx.category}</div>
                        </td>
                        <td className={`px-6 py-4 font-semibold ${tx.type === TransactionType.INCOME ? 'text-emerald-600' : 'text-rose-600'}`}>
                          {tx.type === TransactionType.INCOME ? '+' : '-'}{tx.amount.toLocaleString()}
                        </td>
                        <td className="px-6 py-4 font-bold text-slate-700">
                          ৳ {toBDT(tx.amount)}
                        </td>
                        <td className="px-6 py-4 text-right">
                          <button 
                            onClick={() => deleteTransaction(tx.id)}
                            className="text-slate-300 hover:text-rose-500 p-1 opacity-0 group-hover:opacity-100 transition-all"
                          >
                            <svg className="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                              <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" />
                            </svg>
                          </button>
                        </td>
                      </tr>
                    ))
                  )}
                </tbody>
              </table>
            </div>
          </div>
        </div>
      </div>
    </div>
  );
};

export default App;
