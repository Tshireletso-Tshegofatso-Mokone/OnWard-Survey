import React, { useState, useEffect } from 'react';
import Header from './components/Header';
import SurveyForm from './components/SurveyForm';
import AdminLoginModal from './components/AdminLoginModal';
import AdminPanel from './components/AdminPanel';
import { apiService } from './data/apiService';
import { auth } from './firebase/config';
import BackgroundSlideshow from './components/BackgroundSlideshow';

type View = 'survey' | 'adminLogin' | 'adminPanel';

const App: React.FC = () => {
  const [view, setView] = useState<View>('survey');
  const [isAdminLoggedIn, setIsAdminLoggedIn] = useState(false);
  const dataMode = apiService.getMode();

  useEffect(() => {
    // If Firebase auth is not configured, do nothing.
    if (!auth) {
        return;
    }

    // Listen for authentication state changes
    const unsubscribe = auth.onAuthStateChanged((user: any) => {
      if (user) {
        setIsAdminLoggedIn(true);
        setView('adminPanel'); // Go to admin panel if logged in
      } else {
        setIsAdminLoggedIn(false);
        // If the user was on the admin panel and logs out, redirect to survey
        if (view === 'adminPanel') {
            setView('survey');
        }
      }
    });
    
    // Cleanup subscription on unmount
    return () => unsubscribe();
  }, [view]);


  const handleShowAdminLogin = () => {
    setView('adminLogin');
  };

  const handleLogout = () => {
    if (auth) {
        auth.signOut().catch((error: any) => {
            console.error("Logout failed:", error);
            alert("Logout failed. Please try again.");
        });
    }
    // The onAuthStateChanged listener will handle the view change
  };

  const handleCancelLogin = () => {
    setView('survey');
  };

  return (
    <div className="relative min-h-screen bg-gray-800 text-[#122] font-sans">
      <BackgroundSlideshow />
      <div className="relative z-10 max-w-4xl mx-auto p-4 md:p-8">
        <Header onAdminLoginClick={handleShowAdminLogin} />
        <main className="mt-4">
          {view === 'survey' && <SurveyForm onSubmit={apiService.saveResponse} />}
          {view === 'adminLogin' && (
            <AdminLoginModal onCancel={handleCancelLogin} />
          )}
          {view === 'adminPanel' && isAdminLoggedIn && <AdminPanel onLogout={handleLogout} getResponses={apiService.getResponses} dataMode={dataMode} />}
        </main>
      </div>
    </div>
  );
};

export default App;

import React, { useState } from 'react';
import Card from './Card';
import { auth, isFirebaseConfigured } from '../firebase/config';

interface AdminLoginModalProps {
  onCancel: () => void;
}

const AdminLoginModal: React.FC<AdminLoginModalProps> = ({ onCancel }) => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [isLoading, setIsLoading] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!email.trim() || !password) {
      setError('Please enter your email and password.');
      return;
    }
    
    setError('');
    setIsLoading(true);

    try {
        await auth.signInWithEmailAndPassword(email, password);
        onCancel(); // Close modal on success. Auth listener in App.tsx will redirect.
    } catch (err: any) {
        // This handles modern Firebase Auth errors, including the generic
        // 'INVALID_LOGIN_CREDENTIALS' which is sent to prevent email enumeration.
        // The v8 SDK might not map this error code, so we check the message as a fallback.
        const code = err.code;
        const message = err.message || '';

        if (
            code === 'auth/user-not-found' ||
            code === 'auth/wrong-password' ||
            code === 'auth/invalid-credential' || // For future SDK upgrades
            message.includes('INVALID_LOGIN_CREDENTIALS')
        ) {
            setError('Invalid email or password.');
        } else if (code === 'auth/invalid-email') {
            setError('Please enter a valid email address.');
        } else {
            setError('An unexpected error occurred. Please try again.');
            console.error("Auth error:", err);
        }
    } finally {
        setIsLoading(false);
    }
  };

  if (!isFirebaseConfigured || !auth) {
    return (
        <Card>
            <h2 className="text-2xl font-bold text-[#1f3b73]">Admin Login Unavailable</h2>
            <p className="text-gray-600 mt-2">Firebase has not been configured, so live authentication is disabled. Please update the <code>firebase/config.ts</code> file to enable this feature.</p>
            <div className="mt-4">
                <button type="button" onClick={onCancel} className="bg-[#eef6ff] border border-[#d7e8ff] text-[#1f3b73] font-bold py-2.5 px-4 rounded-lg hover:bg-[#dbe9fa] transition-colors">Go Back</button>
            </div>
        </Card>
    );
  }

  return (
    <Card>
      <h2 className="text-2xl font-bold text-[#1f3b73]">Admin Login</h2>
      <p className="text-sm text-gray-600 mt-1">Access the admin panel to view survey responses.</p>
      <form onSubmit={handleSubmit} className="mt-4">
        <div>
          <label htmlFor="adminEmail" className="block font-semibold text-[#2b4a6f]">Email</label>
          <input
            id="adminEmail"
            name="adminEmail"
            type="email"
            required
            value={email}
            onChange={(e) => setEmail(e.target.value)}
            className="w-full p-2.5 rounded-lg border border-[#d8e0ec] mt-2 bg-white focus:ring-2 focus:ring-[#ff7f7f] focus:border-[#ff7f7f] outline-none"
          />
        </div>
        <div className="mt-4">
          <label htmlFor="adminPass" className="block font-semibold text-[#2b4a6f]">Password</label>
          <input
            id="adminPass"
            name="adminPass"
            type="password"
            required
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            className="w-full p-2.5 rounded-lg border border-[#d8e0ec] mt-2 bg-white focus:ring-2 focus:ring-[#ff7f7f] focus:border-[#ff7f7f] outline-none"
          />
        </div>
        
        {error && <div className="mt-3 text-sm text-red-600 font-medium p-2 bg-red-50 rounded-md">{error}</div>}
        
        <div className="mt-6 flex flex-col sm:flex-row gap-3">
          <button type="submit" disabled={isLoading} className="bg-[#ff7f7f] text-white font-bold py-2.5 px-4 rounded-lg hover:bg-[#e63946] transition-colors disabled:bg-gray-400">
            {isLoading ? 'Processing...' : 'Login'}
          </button>
          <button type="button" onClick={onCancel} disabled={isLoading} className="bg-[#eef6ff] border border-[#d7e8ff] text-[#1f3b73] font-bold py-2.5 px-4 rounded-lg hover:bg-[#dbe9fa] transition-colors disabled:opacity-50">
            Cancel
          </button>
        </div>
      </form>
    </Card>
  );
};

export default AdminLoginModal;

import React, { useState, useEffect, useCallback, useRef } from 'react';
import type { SurveyResponse } from '../types';
import Card from './Card';

// To inform TypeScript that Chart.js is loaded globally from a CDN
declare const Chart: any;

interface AdminPanelProps {
  onLogout: () => void;
  getResponses: () => Promise<SurveyResponse[]>;
  dataMode: 'local' | 'live';
}

const AdminPanel: React.FC<AdminPanelProps> = ({ onLogout, getResponses, dataMode }) => {
  const [responses, setResponses] = useState<SurveyResponse[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  
  const roleChartRef = useRef<HTMLCanvasElement>(null);
  const paymentChartRef = useRef<HTMLCanvasElement>(null);
  const roleChartInstance = useRef<any>(null);
  const paymentChartInstance = useRef<any>(null);


  const fetchResponses = useCallback(async () => {
    setIsLoading(true);
    setError(null);
    try {
        const storedResponses = await getResponses();
        setResponses(dataMode === 'local' ? storedResponses.slice().reverse() : storedResponses);
    } catch(error) {
        console.error("Failed to fetch responses:", error);
        const errorMessage = error instanceof Error ? error.message : "Could not load responses.";
        setError(errorMessage);
    } finally {
        setIsLoading(false);
    }
  }, [getResponses, dataMode]);

  useEffect(() => {
    fetchResponses();
  }, [fetchResponses]);
  
  useEffect(() => {
    if (responses.length > 0) {
      if (roleChartInstance.current) {
        roleChartInstance.current.destroy();
      }
      if (paymentChartInstance.current) {
        paymentChartInstance.current.destroy();
      }
      
      if (roleChartRef.current) {
        const roleCtx = roleChartRef.current.getContext('2d');
        const roleCounts = responses.reduce((acc, res) => {
          const role = res.role || 'Unknown';
          acc[role] = (acc[role] || 0) + 1;
          return acc;
        }, {} as Record<string, number>);

        const roleLabels = Object.keys(roleCounts).map(r => r.replace(/_/g, ' ').replace(/\b\w/g, l => l.toUpperCase()));

        roleChartInstance.current = new Chart(roleCtx, {
          type: 'bar',
          data: {
            labels: roleLabels,
            datasets: [{
              label: '# of Responses',
              data: Object.values(roleCounts),
              backgroundColor: '#ff7f7f',
              borderColor: '#e63946',
              borderWidth: 1
            }]
          },
          options: {
            responsive: true,
            maintainAspectRatio: false,
            plugins: {
                title: { display: true, text: 'Role Distribution', font: { size: 16 }, color: '#1f3b73' },
                legend: { display: false }
            },
            scales: {
              y: { beginAtZero: true, ticks: { stepSize: 1 } }
            }
          }
        });
      }
      
      if (paymentChartRef.current) {
        const paymentCtx = paymentChartRef.current.getContext('2d');
        const paymentCounts = responses.reduce((acc, res) => {
          if (res.paymentModel) {
            acc[res.paymentModel] = (acc[res.paymentModel] || 0) + 1;
          }
          return acc;
        }, {} as Record<string, number>);
        
        const paymentLabels = Object.keys(paymentCounts).map(p => p.charAt(0).toUpperCase() + p.slice(1));
        
        paymentChartInstance.current = new Chart(paymentCtx, {
          type: 'doughnut',
          data: {
            labels: paymentLabels,
            datasets: [{
              label: 'Payment Models',
              data: Object.values(paymentCounts),
              backgroundColor: ['#ff7f7f', '#1f3b73', '#457b9d', '#a8dadc'],
            }]
          },
          options: {
            responsive: true,
            maintainAspectRatio: false,
            plugins: {
              title: { display: true, text: 'Payment Model Preference', font: { size: 16 }, color: '#1f3b73' },
              legend: { position: 'top' }
            }
          }
        });
      }
    }
    
    return () => {
        if(roleChartInstance.current) roleChartInstance.current.destroy();
        if(paymentChartInstance.current) paymentChartInstance.current.destroy();
    }
  }, [responses]);

  const downloadCSV = () => {
    if (responses.length === 0) {
      alert('No responses yet');
      return;
    }
    
    const allKeys = new Set<string>();
    // When downloading, always show newest first, so reverse the local storage array.
    // The live data already comes in descending order.
    const orderedResponses = dataMode === 'local' ? [...responses].reverse() : responses;
    orderedResponses.forEach(r => Object.keys(r).forEach(k => allKeys.add(k)));
    const headers = Array.from(allKeys);
    
    const rows = orderedResponses.map(r => 
        headers.map(h => {
            const value = r[h];
            if (value === undefined || value === null) return '';
            const strValue = Array.isArray(value) ? value.join('; ') : String(value);
            return `"${strValue.replace(/"/g, '""')}"`;
        }).join(',')
    );

    const csvContent = [headers.join(','), ...rows].join('\n');
    const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' });
    const url = URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.setAttribute('href', url);
    link.setAttribute('download', 'onward_responses.csv');
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
    URL.revokeObjectURL(url);
  };
  
  const getSummary = (response: SurveyResponse): string => {
    const excludedKeys = ['when', 'role', 'paymentModel', 'paymentAmount'];
    return Object.entries(response)
        .filter(([key]) => !excludedKeys.includes(key))
        .slice(0, 4)
        .map(([key, value]) => `${key}: ${Array.isArray(value) ? value.join(', ') : value}`)
        .join(' | ');
  }
  
  const responsesToShow = dataMode === 'local' ? responses : responses;
  const totalResponses = responsesToShow.length;

  return (
    <Card>
      <div className="flex justify-between items-start mb-2">
          <h2 className="text-2xl font-bold text-[#1f3b73]">Admin — Responses</h2>
          <div>
            <div className={`text-xs font-bold py-1 px-3 rounded-full text-center ${dataMode === 'local' ? 'bg-yellow-100 text-yellow-800' : 'bg-green-100 text-green-800'}`} role="status">
                {dataMode === 'local' ? 'Data Mode: Local (Not Live)' : 'Data Mode: Live (Connected)'}
            </div>
            {dataMode === 'local' && (
                <p className="text-xs text-yellow-700 mt-1 text-center">Configure <code>firebase/config.ts</code> to go live.</p>
            )}
          </div>
      </div>
      <div className="flex flex-wrap gap-3 items-center mt-3">
        <button onClick={downloadCSV} className="bg-[#ff7f7f] text-white font-bold py-2.5 px-4 rounded-lg hover:bg-[#e63946] transition-colors">Download Master CSV</button>
        <button onClick={fetchResponses} disabled={isLoading} className="bg-[#eef6ff] border border-[#d7e8ff] text-[#1f3b73] font-bold py-2.5 px-4 rounded-lg hover:bg-[#dbe9fa] transition-colors disabled:bg-gray-300 disabled:cursor-not-allowed">
          {isLoading ? 'Refreshing...' : 'Refresh data'}
        </button>
        <button onClick={onLogout} className="bg-[#eef6ff] border border-[#d7e8ff] text-[#1f3b73] font-bold py-2.5 px-4 rounded-lg hover:bg-[#dbe9fa] transition-colors">Logout</button>
      </div>

      {!isLoading && !error && responses.length > 0 && (
        <div className="mt-8">
            <h3 className="text-xl font-bold text-[#e63946] mb-4">Response Visualizations</h3>
            <div className="grid grid-cols-1 md:grid-cols-2 gap-8">
                <div className="relative h-80 bg-white p-4 rounded-lg border border-gray-200 shadow-sm">
                    <canvas ref={roleChartRef}></canvas>
                </div>
                <div className="relative h-80 bg-white p-4 rounded-lg border border-gray-200 shadow-sm">
                    <canvas ref={paymentChartRef}></canvas>
                </div>
            </div>
        </div>
      )}

      <h3 className="text-xl font-bold text-[#e63946] mt-8">Recent responses</h3>
      <div className="mt-3 min-h-[10rem] overflow-auto border rounded-lg">
        <table className="w-full text-sm text-left text-gray-600">
          <thead className="text-xs text-gray-700 uppercase bg-gray-50 sticky top-0">
            <tr>
              <th scope="col" className="px-4 py-3">#</th>
              <th scope="col" className="px-4 py-3">Role</th>
              <th scope="col" className="px-4 py-3">Summary</th>
              <th scope="col" className="px-4 py-3">Payment</th>
              <th scope="col" className="px-4 py-3">When</th>
            </tr>
          </thead>
          <tbody>
            {isLoading && (
              <tr>
                <td colSpan={5} className="text-center py-12 text-gray-500">Loading responses...</td>
              </tr>
            )}
            {error && !isLoading && (
              <tr>
                <td colSpan={5} className="text-center py-12 text-red-600 font-medium">Error: {error}</td>
              </tr>
            )}
            {!isLoading && !error && responses.length === 0 && (
                <tr>
                    <td colSpan={5} className="text-center py-12 text-gray-500">No responses yet.</td>
                </tr>
            )}
            {!isLoading && !error && responsesToShow.map((res, index) => (
              <tr key={res.when + index} className="bg-white border-b hover:bg-gray-50">
                <td className="px-4 py-3 font-medium text-gray-900">{totalResponses - index}</td>
                <td className="px-4 py-3 capitalize">{res.role?.replace(/_/g, ' ')}</td>
                <td className="px-4 py-3 max-w-xs truncate">{getSummary(res)}</td>
                <td className="px-4 py-3">{res.paymentModel}{res.paymentAmount ? ` - ${res.paymentAmount}` : ''}</td>
                <td className="px-4 py-3 whitespace-nowrap">{new Date(res.when).toLocaleString()}</td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </Card>
  );
};

export default AdminPanel;

import React, { useState, useEffect } from 'react';

// A new, curated collection of 10 high-quality images representing the entire logistics industry.
const images = [
  'https://images.pexels.com/photos/2225499/pexels-photo-2225499.jpeg?auto=compress&cs=tinysrgb&w=1260&h=750&dpr=2', // 1. Close-up of ship containers
  'https://images.pexels.com/photos/3807755/pexels-photo-3807755.jpeg?auto=compress&cs=tinysrgb&w=1260&h=750&dpr=2', // 2. Warehouse with shelves and forklift
  'https://images.pexels.com/photos/4483775/pexels-photo-4483775.jpeg?auto=compress&cs=tinysrgb&w=1260&h=750&dpr=2', // 3. Aerial of port with cranes
  'https://images.pexels.com/photos/1117210/pexels-photo-1117210.jpeg?auto=compress&cs=tinysrgb&w=1260&h=750&dpr=2', // 4. Truck with a shipping container on the road
  'https://images.pexels.com/photos/4614165/pexels-photo-4614165.jpeg?auto=compress&cs=tinysrgb&w=1260&h=750&dpr=2', // 5. Cargo plane
  'https://images.pexels.com/photos/777059/pexels-photo-777059.jpeg?auto=compress&cs=tinysrgb&w=1260&h=750&dpr=2', // 6. Port with vast stacks of containers
  'https://images.pexels.com/photos/1572386/pexels-photo-1572386.jpeg?auto=compress&cs=tinysrgb&w=1260&h=750&dpr=2', // 7. Freight train from above
  'https://images.pexels.com/photos/5666420/pexels-photo-5666420.jpeg?auto=compress&cs=tinysrgb&w=1260&h=750&dpr=2', // 8. Two workers with tablet in modern warehouse
  'https://images.pexels.com/photos/1203803/pexels-photo-1203803.jpeg?auto=compress&cs=tinysrgb&w=1260&h=750&dpr=2', // 9. Large container ship on the ocean
  'https://images.pexels.com/photos/4246234/pexels-photo-4246234.jpeg?auto=compress&cs=tinysrgb&w=1260&h=750&dpr=2'  // 10. Truck carrying container leaving a port
];

const BackgroundSlideshow: React.FC = () => {
  const [currentIndex, setCurrentIndex] = useState(0);

  useEffect(() => {
    const intervalId = setInterval(() => {
      setCurrentIndex(prevIndex => (prevIndex + 1) % images.length);
    }, 7000); // Change image every 7 seconds

    return () => clearInterval(intervalId);
  }, []);

  return (
    <div className="absolute inset-0 w-full h-full z-0" aria-hidden="true">
      {images.map((image, index) => (
        <div
          key={index}
          className="absolute inset-0 w-full h-full bg-cover bg-center transition-opacity duration-2000 ease-in-out"
          style={{
            backgroundImage: `url(${image})`,
            opacity: index === currentIndex ? 1 : 0,
            // The new, foolproof method: apply a filter directly to the image
            filter: 'brightness(0.6) sepia(0.2)',
          }}
        />
      ))}
    </div>
  );
};

export default BackgroundSlideshow;

import React from 'react';

interface HeaderProps {
  onAdminLoginClick: () => void;
}

const Header: React.FC<HeaderProps> = ({ onAdminLoginClick }) => {
  return (
    <header className="flex items-center justify-between py-4 border-b-4 border-[#ff7f7f]">
      <div>
        <div className="font-extrabold text-[#1f3b73] text-2xl">OnWard</div>
        <div className="text-sm text-[#e63946] font-semibold">Global-Trade-Specialist</div>
      </div>
      <div>
        <button
          onClick={onAdminLoginClick}
          className="bg-transparent border border-blue-900/10 py-2 px-3 rounded-lg text-[#1f3b73] cursor-pointer hover:bg-blue-50 transition-colors"
        >
          Admin Login
        </button>
      </div>
    </header>
  );
};

export default Header;

import React, { useState, useEffect } from 'react';
import type { Role, SurveyResponse, Question } from '../types';
import { ROLE_QUESTIONS, REQUIRED_MAP, FRIENDLY_LABELS } from '../constants';
import Card from './Card';

const initialFormData = {
  role: '' as Role,
  paymentModel: '',
  paymentAmount: '',
};

interface SurveyFormProps {
  onSubmit: (response: SurveyResponse) => Promise<any>;
}

const QuestionRenderer: React.FC<{ question: Question, formData: any, handleChange: (id: string, value: string | string[], type: string) => void }> = ({ question, formData, handleChange }) => {
  const value = formData[question.id] || (question.type === 'checkbox' ? [] : '');

  return (
    <div key={question.id} className="mt-4">
      <label className="block font-semibold text-[#2b4a6f]">{question.label}</label>
      {question.hint && <p className="text-sm text-[#475569] mb-2">{question.hint}</p>}
      {question.type === 'select' && (
        <select
          id={question.id}
          name={question.id}
          value={value}
          onChange={(e) => handleChange(question.id, e.target.value, 'select')}
          className="w-full p-2.5 rounded-lg border border-[#d8e0ec] mt-2 bg-white focus:ring-2 focus:ring-[#ff7f7f] focus:border-[#ff7f7f] outline-none"
        >
          {question.options.map(opt => <option key={opt.value} value={opt.value}>{opt.label}</option>)}
        </select>
      )}
      {question.type === 'checkbox' && (
        <div className="mt-2 space-y-2">
          {question.options.map(opt => (
            <label key={opt.value} className="flex items-center font-normal text-gray-700">
              <input
                type="checkbox"
                name={question.id}
                value={opt.value}
                checked={value.includes(opt.value)}
                onChange={(e) => {
                  const newValues = e.target.checked
                    ? [...value, e.target.value]
                    : value.filter((v: string) => v !== e.target.value);
                  handleChange(question.id, newValues, 'checkbox');
                }}
                className="mr-2 h-4 w-4 rounded border-gray-300 text-[#ff7f7f] focus:ring-[#ff7f7f]"
              />
              {opt.label}
            </label>
          ))}
        </div>
      )}
    </div>
  );
};


const SurveyForm: React.FC<SurveyFormProps> = ({ onSubmit }) => {
  const [page, setPage] = useState(1);
  const [formData, setFormData] = useState<any>(initialFormData);
  const [showThankYou, setShowThankYou] = useState(false);
  const [isSubmitting, setIsSubmitting] = useState(false);

  const selectedRole = formData.role as Exclude<Role, ''>;

  const handleInputChange = (id: string, value: string | string[], type: string) => {
    setFormData((prev: any) => ({ ...prev, [id]: value }));
  };
  
  const handleRoleChange = (e: React.ChangeEvent<HTMLSelectElement>) => {
    const newRole = e.target.value as Role;
    const oldRole = formData.role;
    let newFormData: any = { ...initialFormData, role: newRole, paymentModel: formData.paymentModel, paymentAmount: formData.paymentAmount };
    if (oldRole && oldRole !== newRole && ROLE_QUESTIONS[oldRole]) {
        const oldQuestions = [...ROLE_QUESTIONS[oldRole].part1, ...ROLE_QUESTIONS[oldRole].part2];
        const currentData: any = { ...formData };
        oldQuestions.forEach(q => delete currentData[q.id]);
        newFormData = { ...currentData, role: newRole };
    }
    setFormData(newFormData);
  };
  
  const validatePage = (pageNum: number): boolean => {
    if (!selectedRole) {
        if (pageNum > 1) {
            alert('Please select a role to continue.');
            setPage(1);
        }
      return false;
    }
    const partKey = pageNum === 1 ? 'part1' : 'part2';
    const requiredGroups = REQUIRED_MAP[selectedRole]?.[partKey] || [];
    const missing: string[] = [];

    requiredGroups.forEach((groupName: string) => {
      const value = formData[groupName];
      const hasValue = Array.isArray(value) ? value.length > 0 : !!value;
      if (!hasValue) {
        missing.push(FRIENDLY_LABELS[groupName] || groupName);
      }
    });

    if (missing.length > 0) {
      alert(`Please complete required fields: ${missing.join(', ')}`);
      return false;
    }
    return true;
  };

  const changePage = (newPage: number) => {
    if (newPage > page) { // Moving forward
        if (page === 1 && !formData.role) {
            alert('Please select a role to continue.');
            return;
        }
        if (page === 1 || page === 2) {
            if (!validatePage(page)) return;
        }
    }
    setPage(newPage);
    window.scrollTo({ top: 0, behavior: 'smooth' });
  };
  
  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!validatePage(1) || !validatePage(2)) {
        if (!validatePage(1)) setPage(1);
        else setPage(2);
        return;
    }
    
    const pm = formData.paymentModel;
    if(!pm){ alert('Please select a payment model'); setPage(3); return }
    if((pm==='subscription' || pm==='annual' || pm==='freemium') && !formData.paymentAmount){ alert('Please select an amount'); setPage(3); return }

    setIsSubmitting(true);
    const response: SurveyResponse = {
      ...formData,
      when: new Date().toISOString(),
    };

    try {
      await onSubmit(response);
      setShowThankYou(true);
      setFormData(initialFormData);
      setPage(1);
    } catch (error) {
        console.error("Submission failed:", error);
        alert(error instanceof Error ? error.message : "Sorry, there was an error saving your response.");
    } finally {
        setIsSubmitting(false);
    }
  };
  
  useEffect(() => {
    if(showThankYou) {
        const timer = setTimeout(() => setShowThankYou(false), 6000);
        return () => clearTimeout(timer);
    }
  }, [showThankYou])

  return (
    <Card>
      {showThankYou && (
        <div className="p-4 mb-4 text-green-800 bg-green-100 border-l-4 border-green-500 rounded-md" role="alert">
          <p className="font-bold">✅ Thank you for your cooperation.</p>
        </div>
      )}
      <form onSubmit={handleSubmit}>
        <div className={page === 1 ? 'block' : 'hidden'}>
          <h2 className="text-2xl font-bold text-[#1f3b73]">Step 1 — Tell us who you are</h2>
          <p className="text-sm text-gray-500 mt-1">Choose one role — we'll show specific questions for that role.</p>
          <div className="mt-4">
            <label htmlFor="role" className="block font-semibold text-[#2b4a6f]">Which of these best describes your role? *</label>
            <select id="role" name="role" value={selectedRole} onChange={handleRoleChange} required
                className="w-full p-2.5 rounded-lg border border-[#d8e0ec] mt-2 bg-white focus:ring-2 focus:ring-[#ff7f7f] focus:border-[#ff7f7f] outline-none">
              <option value="">-- Select --</option>
              <option value="seller">Seller</option>
              <option value="transporter">Transporter</option>
              <option value="warehouse">Warehouse / Factory</option>
              <option value="freight_forwarder">Freight Forwarder</option>
              <option value="importer_exporter">Importer / Exporter</option>
              <option value="distributor">Distributor</option>
            </select>
          </div>
          {selectedRole && ROLE_QUESTIONS[selectedRole]?.part1.map(q => <QuestionRenderer key={q.id} question={q} formData={formData} handleChange={handleInputChange} />)}
          <div className="mt-6">
            <button type="button" onClick={() => changePage(2)} className="w-full sm:w-auto bg-[#ff7f7f] text-white font-bold py-3 px-5 rounded-lg hover:bg-[#e63946] transition-colors">Next — more about your role</button>
          </div>
        </div>

        <div className={page === 2 ? 'block' : 'hidden'}>
          <h2 className="text-2xl font-bold text-[#1f3b73]">Step 2 — A few more role-specific questions</h2>
          <p className="text-sm text-gray-500 mt-1">Answer the remaining questions for your role.</p>
          {selectedRole && ROLE_QUESTIONS[selectedRole]?.part2.map(q => <QuestionRenderer key={q.id} question={q} formData={formData} handleChange={handleInputChange} />)}
          <div className="mt-6 flex gap-3">
            <button type="button" onClick={() => changePage(3)} className="bg-[#ff7f7f] text-white font-bold py-3 px-5 rounded-lg hover:bg-[#e63946] transition-colors">Next — payments</button>
            <button type="button" onClick={() => changePage(1)} className="bg-[#eef6ff] border border-[#d7e8ff] text-[#1f3b73] font-bold py-3 px-5 rounded-lg hover:bg-[#dbe9fa] transition-colors">Back</button>
          </div>
        </div>

        <div className={page === 3 ? 'block' : 'hidden'}>
          <h2 className="text-2xl font-bold text-[#1f3b73]">Step 3 — Payment model & pricing</h2>
          <p className="text-sm text-gray-500 mt-1">Select how you'd prefer to pay for services on OnWard (amounts in ZAR).</p>
          <div className="mt-4">
            <label htmlFor="paymentModel" className="block font-semibold text-[#2b4a6f]">Preferred payment model *</label>
            <select id="paymentModel" name="paymentModel" value={formData.paymentModel} onChange={e => handleInputChange('paymentModel', e.target.value, 'select')} required
                className="w-full p-2.5 rounded-lg border border-[#d8e0ec] mt-2 bg-white focus:ring-2 focus:ring-[#ff7f7f] focus:border-[#ff7f7f] outline-none">
                <option value="">-- Select --</option>
                <option value="subscription">Monthly subscription</option>
                <option value="annual">Annual</option>
                <option value="freemium">Freemium</option>
                <option value="integrated">Integrated payment processing (in-app payments)</option>
            </select>
          </div>
          {['subscription', 'annual', 'freemium'].includes(formData.paymentModel) && (
            <div className="mt-4">
                <label htmlFor="paymentAmount" className="block font-semibold text-[#2b4a6f]">If subscription/annual/freemium, how much would you pay per month? *</label>
                <select id="paymentAmount" name="paymentAmount" value={formData.paymentAmount} onChange={e => handleInputChange('paymentAmount', e.target.value, 'select')} required
                    className="w-full p-2.5 rounded-lg border border-[#d8e0ec] mt-2 bg-white focus:ring-2 focus:ring-[#ff7f7f] focus:border-[#ff7f7f] outline-none">
                    <option value="">-- Select --</option>
                    <option value="R0-R500">R0 – R500</option>
                    <option value="R500-R1000">R500 – R1 000</option>
                    <option value="R1000-R2500">R1 000 – R2 500</option>
                    <option value="R2500-R5000">R2 500 – R5 000</option>
                    <option value="R5000+">R5 000+</option>
                </select>
            </div>
          )}
          {formData.paymentModel === 'integrated' && (
             <div className="mt-3 p-3 bg-blue-50 border-l-4 border-blue-400 text-sm text-blue-800 rounded-r-md">
                Choosing <strong>Integrated payment processing</strong> means sellers can accept payments inside the OnWard platform. (We may charge a small commission per transaction.)
             </div>
          )}
          <div className="mt-6 flex gap-3">
            <button type="submit" disabled={isSubmitting} className="bg-[#ff7f7f] text-white font-bold py-3 px-5 rounded-lg hover:bg-[#e63946] transition-colors disabled:bg-gray-400 disabled:cursor-not-allowed">
              {isSubmitting ? 'Submitting...' : 'Submit survey'}
            </button>
            <button type="button" onClick={() => changePage(2)} disabled={isSubmitting} className="bg-[#eef6ff] border border-[#d7e8ff] text-[#1f3b73] font-bold py-3 px-5 rounded-lg hover:bg-[#dbe9fa] transition-colors disabled:opacity-50">
              Back
            </button>
          </div>
        </div>
      </form>
    </Card>
  );
};

export default SurveyForm;


import type { AllRoleQuestions } from './types';

// Structured questions for easier rendering in React
export const ROLE_QUESTIONS: AllRoleQuestions = {
  seller: {
    part1: [
      { id: 'seller_product_types', label: 'What type of products do you sell? (select at least one)', type: 'checkbox', hint: 'Products & channels', options: [{ value: 'Handmade', label: 'Handmade' }, { value: 'Manufactured', label: 'Manufactured' }, { value: 'Resale', label: 'Resale' }, { value: 'Digital', label: 'Digital products' }] },
      { id: 'seller_channels', label: 'Where do you currently sell? (select at least one)', type: 'checkbox', options: [{ value: 'Local', label: 'Local only' }, { value: 'OnlineDomestic', label: 'Online (domestic)' }, { value: 'OnlineInternational', label: 'Online (international)' }, { value: 'InPerson', label: 'In-person markets/events' }] }
    ],
    part2: [
      { id: 'seller_export_experience', label: 'Export experience', type: 'select', options: [{ value: '', label: '-- Select --' }, { value: 'None', label: 'None' }, { value: 'Some', label: 'Some' }, { value: 'Regular', label: 'Regular exports' }] },
      { id: 'seller_challenges', label: 'Biggest challenges when selling to other countries (select at least one)', type: 'checkbox', options: [{ value: 'FindingBuyers', label: 'Finding buyers' }, { value: 'ShippingCosts', label: 'Shipping costs' }, { value: 'Customs', label: 'Customs paperwork' }, { value: 'Payments', label: 'Payment solutions' }, { value: 'Marketing', label: 'Marketing and promotion' }] },
      { id: 'seller_services', label: 'Which services would you be interested in? (select at least one)', type: 'checkbox', options: [{ value: 'Marketplace', label: 'Marketplace listing' }, { value: 'Photography', label: 'Product photography' }, { value: 'Translation', label: 'Translation services' }, { value: 'PaymentSolutions', label: 'Payment solutions' }, { value: 'BulkShipping', label: 'Bulk shipping discounts' }] }
    ]
  },
  transporter: {
    part1: [
      { id: 'trans_types', label: 'What type(s) of transport do you offer? (select at least one)', type: 'checkbox', options: [{ value: 'Road', label: 'Road' }, { value: 'Rail', label: 'Rail' }, { value: 'Air', label: 'Air' }, { value: 'Sea', label: 'Sea' }] },
      { id: 'trans_capacity', label: 'Carrying capacity (select at least one)', type: 'checkbox', options: [{ value: 'SmallPackages', label: 'Small packages' }, { value: 'Pallets', label: 'Pallets' }, { value: 'Containers', label: 'Containers' }, { value: 'Bulk', label: 'Bulk freight' }] }
    ],
    part2: [
      { id: 'trans_routes', label: 'Current routes (select at least one)', type: 'checkbox', options: [{ value: 'Local', label: 'Local only' }, { value: 'Regional', label: 'Cross-border (regional)' }, { value: 'Global', label: 'Global' }] },
      { id: 'trans_challenges', label: 'Challenges (select at least one)', type: 'checkbox', options: [{ value: 'CustomsDelays', label: 'Customs delays' }, { value: 'RouteOptimization', label: 'Route optimization' }, { value: 'FuelCosts', label: 'Fuel costs' }, { value: 'FindingClients', label: 'Finding clients' }, { value: 'TrackingTech', label: 'Tracking technology' }] },
      { id: 'trans_services', label: 'Which services would you be interested in? (select at least one)', type: 'checkbox', options: [{ value: 'ClientMatching', label: 'Client matching' }, { value: 'RouteSoftware', label: 'Route optimization software' }, { value: 'BulkFuel', label: 'Bulk fuel discounts' }, { value: 'CustomsClearance', label: 'Customs clearance' }, { value: 'TrackingIntegration', label: 'Tracking integration' }] }
    ]
  },
  warehouse: {
    part1: [
      { id: 'wh_type', label: 'Type of facility (select at least one)', type: 'checkbox', options: [{ value: 'Storage', label: 'Storage' }, { value: 'Manufacturing', label: 'Manufacturing' }, { value: 'Both', label: 'Both' }] },
      { id: 'wh_industry', label: 'Industry focus (select at least one)', type: 'checkbox', options: [{ value: 'Food', label: 'Food' }, { value: 'Textiles', label: 'Textiles' }, { value: 'Electronics', label: 'Electronics' }, { value: 'Machinery', label: 'Machinery' }] }
    ],
    part2: [
      { id: 'wh_capacity', label: 'Capacity', type: 'select', options: [{ value: '', label: '-- Select --' }, { value: 'Small', label: 'Small' }, { value: 'Medium', label: 'Medium' }, { value: 'Large', label: 'Large' }] },
      { id: 'wh_automation', label: 'Automation level', type: 'select', options: [{ value: '', label: '-- Select --' }, { value: 'Manual', label: 'Manual' }, { value: 'Semi', label: 'Semi-automated' }, { value: 'Full', label: 'Fully automated' }] },
      { id: 'wh_services', label: 'Which services would you be interested in? (select at least one)', type: 'checkbox', options: [{ value: 'ClientMatching', label: 'Client matching' }, { value: 'AutomationUpgrades', label: 'Automation upgrades' }, { value: 'ExportPackaging', label: 'Export packaging' }, { value: 'ComplianceSupport', label: 'Compliance support' }, { value: 'GlobalMarketing', label: 'Global marketing' }] }
    ]
  },
  freight_forwarder: {
    part1: [
      { id: 'ff_services', label: 'Which services do you offer? (select at least one)', type: 'checkbox', options: [{ value: 'Air', label: 'Air freight' }, { value: 'Sea', label: 'Sea freight' }, { value: 'Road', label: 'Road freight' }, { value: 'Rail', label: 'Rail freight' }, { value: 'CustomsClearance', label: 'Customs clearance' }, { value: 'Warehousing', label: 'Warehousing' }, { value: 'Documentation', label: 'Documentation handling' }] }
    ],
    part2: [
      { id: 'ff_regions', label: 'Regions served (select at least one)', type: 'checkbox', options: [{ value: 'Domestic', label: 'Domestic only' }, { value: 'Regional', label: 'Regional' }, { value: 'Continental', label: 'Continental' }, { value: 'Global', label: 'Global' }] },
      { id: 'ff_avg_shipment', label: 'Average shipment size', type: 'select', options: [{ value: '', label: '-- Select --' }, { value: 'SmallPackages', label: 'Small packages' }, { value: 'Pallets', label: 'Pallets' }, { value: 'Containers', label: 'Containers' }, { value: 'Bulk', label: 'Bulk freight' }] },
      { id: 'ff_challenges', label: 'Challenges (select at least one)', type: 'checkbox', options: [{ value: 'CustomsDelays', label: 'Customs delays' }, { value: 'CarrierCapacity', label: 'Carrier capacity' }, { value: 'DocErrors', label: 'Documentation errors' }, { value: 'FindingClients', label: 'Finding clients' }] },
      { id: 'ff_interested', label: 'Services interested in (select at least one)', type: 'checkbox', options: [{ value: 'ClientAcquisition', label: 'Client acquisition' }, { value: 'TrackingIntegration', label: 'Tracking integration' }, { value: 'TradeCompliance', label: 'Trade compliance' }, { value: 'DigitalDocs', label: 'Digital documentation' }] }
    ]
  },
  importer_exporter: {
    part1: [
      { id: 'imp_exp_business_type', label: 'Business type', type: 'select', options: [{ value: '', label: '-- Select --' }, { value: 'Importer', label: 'Importer' }, { value: 'Exporter', label: 'Exporter' }, { value: 'Both', label: 'Both' }] },
      { id: 'imp_exp_goods', label: 'Type of goods (select at least one)', type: 'checkbox', options: [{ value: 'RawMaterials', label: 'Raw materials' }, { value: 'Agricultural', label: 'Agricultural' }, { value: 'Manufactured', label: 'Manufactured' }, { value: 'Machinery', label: 'Machinery / equipment' }, { value: 'Consumer', label: 'Consumer products' }] }
    ],
    part2: [
      { id: 'imp_exp_regions', label: 'Where do you currently trade? (select all continents that apply)', type: 'checkbox', options: [{ value: 'Africa', label: 'Africa' }, { value: 'Asia', label: 'Asia' }, { value: 'Europe', label: 'Europe' }, { value: 'NorthAmerica', label: 'North America' }, { value: 'SouthAmerica', label: 'South America' }, { value: 'Oceania', label: 'Oceania' }] },
      { id: 'imp_exp_challenges', label: 'Challenges (select at least one)', type: 'checkbox', options: [{ value: 'HighLogisticsCosts', label: 'High logistics costs' }, { value: 'FindingPartners', label: 'Finding reliable partners' }, { value: 'Customs', label: 'Customs clearance' }, { value: 'PaymentCollection', label: 'Payment collection' }] },
      { id: 'imp_exp_services', label: 'Helpful services (select at least one)', type: 'checkbox', options: [{ value: 'MarketResearch', label: 'Market research / buyer matching' }, { value: 'ShippingBooking', label: 'Shipping / freight booking' }, { value: 'CustomsSupport', label: 'Customs compliance support' }, { value: 'TradeFinance', label: 'Trade finance / payment facilitation' }, { value: 'Warehousing', label: 'Warehousing and distribution' }, { value: 'Insurance', label: 'Insurance services' }] }
    ]
  },
  distributor: {
    part1: [
      { id: 'dist_type', label: 'Type of distribution', type: 'checkbox', options: [{ value: 'Wholesale', label: 'Wholesale (B2B)' }, { value: 'Retail', label: 'Retail (B2C)' }, { value: 'Both', label: 'Both' }] },
      { id: 'dist_industries', label: 'Industries you serve (select at least one)', type: 'checkbox', options: [{ value: 'Food', label: 'Food & Beverage' }, { value: 'Textiles', label: 'Textiles & Apparel' }, { value: 'Electronics', label: 'Electronics' }, { value: 'Machinery', label: 'Machinery & Industrial goods' }, { value: 'Pharma', label: 'Pharmaceuticals & Healthcare' }] }
    ],
    part2: [
      { id: 'dist_network', label: 'Network size', type: 'select', options: [{ value: '', label: '-- Select --' }, { value: 'Local', label: 'Local' }, { value: 'Regional', label: 'Regional (multi-country)' }, { value: 'Continental', label: 'Continental' }, { value: 'Global', label: 'Global' }] },
      { id: 'dist_challenges', label: 'Challenges (select at least one)', type: 'checkbox', options: [{ value: 'Inventory', label: 'Inventory management' }, { value: 'DeliveryCosts', label: 'Delivery / logistics costs' }, { value: 'ReliableSuppliers', label: 'Finding reliable suppliers' }, { value: 'PaymentCollection', label: 'Payment collection' }] },
      { id: 'dist_services', label: 'Services interested in (select at least one)', type: 'checkbox', options: [{ value: 'InventoryTech', label: 'Inventory management tools' }, { value: 'Logistics', label: 'Logistics partnerships' }, { value: 'TradeFinance', label: 'Trade finance' }, { value: 'DistributionSync', label: 'Distribution integrations' }] }
    ]
  }
};

export const REQUIRED_MAP = {
  seller: { part1: ['seller_product_types','seller_channels'], part2: ['seller_export_experience','seller_challenges','seller_services'] },
  transporter: { part1: ['trans_types','trans_capacity'], part2: ['trans_routes','trans_challenges','trans_services'] },
  warehouse: { part1: ['wh_type','wh_industry'], part2: ['wh_capacity','wh_automation','wh_services'] },
  freight_forwarder: { part1: ['ff_services'], part2: ['ff_regions','ff_avg_shipment','ff_challenges','ff_interested'] },
  importer_exporter: { part1: ['imp_exp_business_type','imp_exp_goods'], part2: ['imp_exp_regions','imp_exp_challenges','imp_exp_services'] },
  distributor: { part1: ['dist_type','dist_industries'], part2: ['dist_network','dist_challenges','dist_services'] }
};

export const FRIENDLY_LABELS: {[key: string]: string} = {
  seller_product_types: 'Product types', seller_channels: 'Selling channels', seller_export_experience: 'Export experience', seller_challenges: 'Seller challenges', seller_services: 'Seller services',
  trans_types: 'Transport types', trans_capacity: 'Carrying capacity', trans_routes: 'Current routes', trans_challenges: 'Transporter challenges', trans_services: 'Transporter services',
  wh_type: 'Facility type', wh_industry: 'Industry focus', wh_capacity: 'Capacity', wh_automation: 'Automation level', wh_services: 'Warehouse services',
  ff_services: 'Freight services', ff_regions: 'Freight regions', ff_avg_shipment: 'Average shipment size', ff_challenges: 'Freight challenges', ff_interested: 'Freight interested services',
  imp_exp_business_type: 'Business type', imp_exp_goods: 'Type of goods', imp_exp_regions: 'Trading regions', imp_exp_challenges: 'Importer/Exporter challenges', imp_exp_services: 'Importer/Exporter services',
  dist_type: 'Distribution type', dist_industries: 'Distributor industries', dist_network: 'Network size', dist_challenges: 'Distributor challenges', dist_services: 'Distributor services'
};

import type { SurveyResponse } from '../types';
import { firestore, isFirebaseConfigured } from '../firebase/config';

// The data mode is now determined automatically based on your Firebase configuration.
const DATA_MODE: 'local' | 'live' = isFirebaseConfigured && firestore ? 'live' : 'local'; 

// --- Local Storage Implementation (Fallback) ---
const getResponsesFromLocalStorage = async (): Promise<SurveyResponse[]> => {
  await new Promise(resolve => setTimeout(resolve, 500)); 
  try {
    const storedResponses = localStorage.getItem('onward_responses');
    return storedResponses ? JSON.parse(storedResponses) : [];
  } catch (error) {
    console.error("Failed to fetch from local storage:", error);
    throw new Error("Could not load responses from local storage.");
  }
};

const saveResponseToLocalStorage = async (response: SurveyResponse): Promise<{ success: boolean }> => {
  await new Promise(resolve => setTimeout(resolve, 800)); 
  try {
    const storedResponses = localStorage.getItem('onward_responses');
    const responses = storedResponses ? JSON.parse(storedResponses) : [];
    responses.push(response);
    localStorage.setItem('onward_responses', JSON.stringify(responses));
    return { success: true };
  } catch (error) {
    console.error("Failed to save to local storage:", error);
    throw new Error("Sorry, there was an error saving your response locally.");
  }
};

// --- Live API Implementation (Firebase Firestore) ---
const getResponsesFromApi = async (): Promise<SurveyResponse[]> => {
    if (!firestore) {
        throw new Error("Firebase is not configured or initialized correctly.");
    }
    
    const responsesCollection = firestore.collection('responses');
    // Fetch responses and order by submission time, newest first.
    const snapshot = await responsesCollection.orderBy('when', 'desc').get();
    
    if (snapshot.empty) {
        return [];
    }
    
    const responses: SurveyResponse[] = [];
    snapshot.forEach((doc: any) => {
        // We can add the document ID if needed in the future, for now just the data.
        responses.push(doc.data() as SurveyResponse);
    });
    return responses;
};

const saveResponseToApi = async (response: SurveyResponse): Promise<{ success: boolean }> => {
    if (!firestore) {
        throw new Error("Firebase is not configured or initialized correctly.");
    }
    
    // Use the 'when' ISO string as a unique document ID to prevent accidental duplicates.
    await firestore.collection('responses').doc(response.when).set(response);
    return { success: true };
};

// --- Exported Service ---
// The service now intelligently chooses the correct functions based on the data mode.
export const apiService = {
  getResponses: DATA_MODE === 'local' ? getResponsesFromLocalStorage : getResponsesFromApi,
  saveResponse: DATA_MODE === 'local' ? saveResponseToLocalStorage : saveResponseToApi,
  getMode: () => DATA_MODE,
};

// This global declaration is needed because Firebase is loaded from a CDN script in index.html
declare const firebase: any;

// We will inject the config into this global window variable using Netlify.
// This tells TypeScript that we expect window.firebaseConfig to exist.
declare global {
  interface Window {
    firebaseConfig: any;
  }
}

// Read the config from the global window object, or use an empty object as a fallback.
const firebaseConfig = window.firebaseConfig || {};

// This check determines if the config was successfully injected by Netlify.
// It checks for a specific, non-secret key like projectId.
export const isFirebaseConfigured = !!firebaseConfig.projectId;

let db: any = null;
let authService: any = null;

if (isFirebaseConfigured) {
  try {
    // Initialize Firebase if not already initialized
    const app = firebase.apps.length ? firebase.app() : firebase.initializeApp(firebaseConfig);
    // Initialize Cloud Firestore and get a reference to the service
    db = firebase.firestore(app);
    // Initialize Firebase Authentication
    authService = firebase.auth(app);
  } catch (e) {
    console.error("Firebase initialization failed:", e);
  }
} else {
  // This warning will appear in the browser console if the Netlify injection fails.
  console.warn("Firebase config not found. The app will run in 'local' mode. Ensure environment variables and snippet injection are configured in Netlify.");
}

// Export the database instance for the apiService to use.
export const firestore = db;
// Export the auth instance
export const auth = authService;

<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>OnWard — Survey + Admin</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
      body {
        font-family: Inter, Segoe UI, Arial, sans-serif;
      }
    </style>
  <script type="importmap">
{
  "imports": {
    "react/": "https://aistudiocdn.com/react@^19.1.1/",
    "react": "https://aistudiocdn.com/react@^19.1.1",
    "react-dom/": "https://aistudiocdn.com/react-dom@^19.1.1/"
  }
}
</script>
</head>
  <body>
    <noscript>You need to enable JavaScript to run this app.</noscript>
    <div id="root"></div>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <!-- Firebase SDK -->
    <script src="https://www.gstatic.com/firebasejs/8.10.1/firebase-app.js"></script>
    <script src="https://www.gstatic.com/firebasejs/8.10.1/firebase-firestore.js"></script>
    <script src="https://www.gstatic.com/firebasejs/8.10.1/firebase-auth.js"></script>
  </body>
</html>


import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';

const rootElement = document.getElementById('root');
if (!rootElement) {
  throw new Error("Could not find root element to mount to");
}

const root = ReactDOM.createRoot(rootElement);
root.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);

{
  "name": "OnWard Survey & Admin",
  "description": "A multi-step survey application for global trade specialists to gather role-specific feedback. Includes a password-protected admin panel to view and download responses.",
  "requestFramePermissions": []
}


export type Role = 'seller' | 'transporter' | 'warehouse' | 'freight_forwarder' | 'importer_exporter' | 'distributor' | '';

export interface SurveyResponse {
  when: string;
  role: Role;
  paymentModel: string;
  paymentAmount: string;
  [key: string]: any;
}

export interface Question {
  id: string;
  label: string;
  type: 'checkbox' | 'select';
  options: { value: string; label: string }[];
  hint?: string;
}

export interface RoleQuestions {
  part1: Question[];
  part2: Question[];
}

export type AllRoleQuestions = {
  [key in Exclude<Role, ''>]: RoleQuestions;
}

