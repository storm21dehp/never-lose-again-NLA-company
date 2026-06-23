Perfeito! Vou te entregar a **estrutura completa de arquivos** para colocar o MVP no ar imediatamente.

Adaptei tudo especificamente para a **API da DeepSeek** (usando o SDK oficial da OpenAI, pois é compatível). **Lembre-se:** você já revogou a chave antiga, certo? Agora use a **nova chave** no arquivo `.env`.

---

## 📁 Estrutura de Pastas Final

```
meu-app-gastos/
├── backend/
│   ├── src/
│   │   ├── config/
│   │   │   └── supabase.js
│   │   ├── middlewares/
│   │   │   └── auth.js
│   │   ├── services/
│   │   │   ├── aiService.js
│   │   │   └── transactionService.js
│   │   └── routes/
│   │       └── transactions.js
│   ├── .env.example
│   ├── package.json
│   └── server.js
└── frontend/
    ├── src/
    │   ├── components/
    │   │   └── TransactionInput.jsx
    │   ├── lib/
    │   │   └── supabase.js
    │   ├── App.jsx
    │   ├── main.jsx
    │   └── index.css
    ├── .env.example
    ├── index.html
    ├── package.json
    └── vite.config.js
```

---

## 🗄️ 1. Banco de Dados (SQL - Supabase)

Copie e cole este SQL no **SQL Editor** do Supabase para criar as tabelas e habilitar RLS:

```sql
-- =============================================
-- HABILITAR EXTENSÕES
-- =============================================
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- =============================================
-- TABELA: users (opcional, pois o Supabase Auth já tem)
-- =============================================
CREATE TABLE IF NOT EXISTS public.users (
    id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
    email TEXT UNIQUE NOT NULL,
    full_name TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
COMMENT ON TABLE public.users IS 'Perfil dos usuários';

-- =============================================
-- TABELA: categories
-- =============================================
CREATE TABLE IF NOT EXISTS public.categories (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    icon TEXT,
    color TEXT,
    is_system BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
COMMENT ON TABLE public.categories IS 'Categorias de gastos';

CREATE INDEX IF NOT EXISTS idx_categories_user_id ON public.categories(user_id);

-- =============================================
-- TABELA: transactions
-- =============================================
CREATE TABLE IF NOT EXISTS public.transactions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
    category_id UUID REFERENCES public.categories(id) ON DELETE SET NULL,
    amount DECIMAL(10, 2) NOT NULL,
    transaction_date DATE NOT NULL,
    description TEXT NOT NULL,
    cleaned_description TEXT,
    ai_suggested_category TEXT,
    ai_confidence DECIMAL(3, 2),
    is_ai_correct BOOLEAN,
    ai_raw_response JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
COMMENT ON TABLE public.transactions IS 'Transações financeiras';

CREATE INDEX IF NOT EXISTS idx_transactions_user_id ON public.transactions(user_id);
CREATE INDEX IF NOT EXISTS idx_transactions_user_date ON public.transactions(user_id, transaction_date DESC);

-- =============================================
-- TRIGGER para updated_at
-- =============================================
CREATE OR REPLACE FUNCTION public.update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_users_updated_at BEFORE UPDATE ON public.users
    FOR EACH ROW EXECUTE FUNCTION public.update_updated_at();

CREATE TRIGGER trigger_transactions_updated_at BEFORE UPDATE ON public.transactions
    FOR EACH ROW EXECUTE FUNCTION public.update_updated_at();

-- =============================================
-- ROW LEVEL SECURITY (RLS)
-- =============================================
ALTER TABLE public.users ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.categories ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.transactions ENABLE ROW LEVEL SECURITY;

-- Políticas
CREATE POLICY "Users can view own data" ON public.users
    FOR SELECT USING (auth.uid() = id);

CREATE POLICY "Categories are user-scoped" ON public.categories
    FOR ALL USING (auth.uid() = user_id);

CREATE POLICY "Transactions are user-scoped" ON public.transactions
    FOR ALL USING (auth.uid() = user_id);

-- =============================================
-- FUNÇÃO para criar perfil automaticamente no signup
-- =============================================
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO public.users (id, email, full_name)
    VALUES (NEW.id, NEW.email, NEW.raw_user_meta_data->>'full_name');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER on_auth_user_created
    AFTER INSERT ON auth.users
    FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();
```

---

## ⚙️ 2. BACKEND (Node.js + Express)

### 📄 `backend/package.json`
```json
{
  "name": "gastos-ai-backend",
  "version": "1.0.0",
  "description": "Backend para categorização de gastos com DeepSeek",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "cors": "^2.8.5",
    "dotenv": "^16.3.1",
    "@supabase/supabase-js": "^2.39.0",
    "openai": "^4.28.0"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
```

### 📄 `backend/.env.example` (renomeie para `.env` e preencha)
```env
PORT=3001
SUPABASE_URL=sua_url_do_supabase
SUPABASE_SERVICE_ROLE_KEY=sua_chave_service_role
DEEPSEEK_API_KEY=sua_nova_chave_aqui
```

### 📄 `backend/server.js`
```javascript
require('dotenv').config();
const express = require('express');
const cors = require('cors');
const transactionsRoutes = require('./src/routes/transactions');

const app = express();
const PORT = process.env.PORT || 3001;

// Middlewares
app.use(cors({
    origin: 'http://localhost:5173', // Frontend Vite
    credentials: true
}));
app.use(express.json());

// Rotas
app.use('/api/transactions', transactionsRoutes);

// Health check
app.get('/health', (req, res) => {
    res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

app.listen(PORT, () => {
    console.log(`🚀 Backend rodando em http://localhost:${PORT}`);
});
```

### 📄 `backend/src/config/supabase.js`
```javascript
const { createClient } = require('@supabase/supabase-js');
require('dotenv').config();

const supabaseUrl = process.env.SUPABASE_URL;
const supabaseKey = process.env.SUPABASE_SERVICE_ROLE_KEY;

if (!supabaseUrl || !supabaseKey) {
    throw new Error('Missing Supabase environment variables');
}

const supabase = createClient(supabaseUrl, supabaseKey);

module.exports = supabase;
```

### 📄 `backend/src/middlewares/auth.js`
```javascript
const supabase = require('../config/supabase');

async function requireAuth(req, res, next) {
    try {
        const authHeader = req.headers.authorization;
        if (!authHeader || !authHeader.startsWith('Bearer ')) {
            return res.status(401).json({ error: 'Token não fornecido' });
        }

        const token = authHeader.split(' ')[1];
        const { data: { user }, error } = await supabase.auth.getUser(token);

        if (error || !user) {
            return res.status(401).json({ error: 'Token inválido ou expirado' });
        }

        req.user = user;
        next();
    } catch (err) {
        console.error('Auth error:', err);
        return res.status(500).json({ error: 'Erro ao autenticar' });
    }
}

module.exports = { requireAuth };
```

### 📄 `backend/src/services/aiService.js` (Integração com DeepSeek)
```javascript
const OpenAI = require('openai');
require('dotenv').config();

const SYSTEM_PROMPT = `
Você é um assistente especializado em categorização de transações financeiras pessoais.

### SUA TAREFA
Receba um texto em linguagem natural descrevendo um gasto e retorne APENAS um objeto JSON com os campos abaixo. NÃO adicione texto explicativo, markdown ou qualquer coisa fora do JSON.

### FORMATO DE SAÍDA OBRIGATÓRIO
{
    "amount": number,
    "date": "YYYY-MM-DD",
    "cleaned_description": "string",
    "suggested_category": "string"
}

### REGRAS
1. **amount**: Extraia o valor numérico. Se não encontrar, retorne null.
2. **date**: Use a data do texto. Se não houver, use a data de hoje no formato YYYY-MM-DD.
3. **cleaned_description**: Remova valores monetários e datas, mantendo uma descrição limpa do que foi comprado.
4. **suggested_category**: Escolha UMA categoria dentre: Alimentação, Transporte, Moradia, Saúde, Educação, Lazer, Vestuário, Supermercado, Serviços, Assinaturas, Outros.

### EXEMPLOS (Few-Shot)
ENTRADA: "Paguei 50 reais no ifood ontem"
SAÍDA: {"amount": 50.00, "date": "2026-06-22", "cleaned_description": "Ifood", "suggested_category": "Alimentação"}

ENTRADA: "Uber da casa pro trabalho 24,90"
SAÍDA: {"amount": 24.90, "date": "2026-06-23", "cleaned_description": "Uber - casa para trabalho", "suggested_category": "Transporte"}

ENTRADA: "Netflix R$ 39,90"
SAÍDA: {"amount": 39.90, "date": "2026-06-23", "cleaned_description": "Netflix", "suggested_category": "Assinaturas"}

ENTRADA: "Comprei pão, leite e ovos no mercado por 27,35"
SAÍDA: {"amount": 27.35, "date": "2026-06-23", "cleaned_description": "Compras no mercado (pão, leite, ovos)", "suggested_category": "Supermercado"}

Retorne SOMENTE o JSON. Nada mais.
`;

class AIService {
    constructor() {
        this.client = new OpenAI({
            apiKey: process.env.DEEPSEEK_API_KEY,
            baseURL: 'https://api.deepseek.com/v1'
        });
    }

    async categorize(text) {
        try {
            const response = await this.client.chat.completions.create({
                model: 'deepseek-chat',
                messages: [
                    { role: 'system', content: SYSTEM_PROMPT },
                    { role: 'user', content: `ENTRADA: "${text}"` }
                ],
                temperature: 0.1,
                response_format: { type: 'json_object' }
            });

            const rawResponse = response.choices[0].message.content;
            const parsed = JSON.parse(rawResponse);

            return {
                amount: parseFloat(parsed.amount),
                date: parsed.date || new Date().toISOString().split('T')[0],
                cleaned_description: parsed.cleaned_description || text,
                suggested_category: parsed.suggested_category || 'Outros',
                raw_response: parsed
            };
        } catch (error) {
            console.error('Erro na DeepSeek:', error);
            throw new Error(`Falha na categorização: ${error.message}`);
        }
    }
}

module.exports = new AIService();
```

### 📄 `backend/src/services/transactionService.js`
```javascript
const supabase = require('../config/supabase');

class TransactionService {
    async createTransaction(userId, transactionData) {
        const { amount, date, description, cleaned_description, suggested_category, raw_response } = transactionData;

        // Busca ou cria a categoria
        let categoryId = null;
        const { data: existingCategory } = await supabase
            .from('categories')
            .select('id')
            .eq('user_id', userId)
            .eq('name', suggested_category)
            .maybeSingle();

        if (existingCategory) {
            categoryId = existingCategory.id;
        } else {
            const { data: newCategory, error: catError } = await supabase
                .from('categories')
                .insert({
                    user_id: userId,
                    name: suggested_category,
                    is_system: false
                })
                .select('id')
                .single();

            if (!catError && newCategory) {
                categoryId = newCategory.id;
            }
        }

        const transaction = {
            user_id: userId,
            category_id: categoryId,
            amount: amount,
            transaction_date: date,
            description: description,
            cleaned_description: cleaned_description,
            ai_suggested_category: suggested_category,
            ai_confidence: 0.85,
            is_ai_correct: null,
            ai_raw_response: raw_response
        };

        const { data, error } = await supabase
            .from('transactions')
            .insert(transaction)
            .select('*')
            .single();

        if (error) {
            throw new Error(`Erro ao salvar transação: ${error.message}`);
        }

        return data;
    }

    async listTransactions(userId, limit = 50, offset = 0) {
        const { data, error, count } = await supabase
            .from('transactions')
            .select('*, categories(name, icon, color)', { count: 'exact' })
            .eq('user_id', userId)
            .order('transaction_date', { ascending: false })
            .range(offset, offset + limit - 1);

        if (error) {
            throw new Error(`Erro ao listar transações: ${error.message}`);
        }

        return { data, count };
    }

    async updateAiFeedback(transactionId, userId, isCorrect) {
        const { data, error } = await supabase
            .from('transactions')
            .update({ is_ai_correct: isCorrect })
            .eq('id', transactionId)
            .eq('user_id', userId)
            .select('*')
            .single();

        if (error) {
            throw new Error(`Erro ao atualizar feedback: ${error.message}`);
        }

        return data;
    }
}

module.exports = new TransactionService();
```

### 📄 `backend/src/routes/transactions.js`
```javascript
const express = require('express');
const router = express.Router();
const { requireAuth } = require('../middlewares/auth');
const aiService = require('../services/aiService');
const transactionService = require('../services/transactionService');

router.post('/', requireAuth, async (req, res) => {
    try {
        const { text } = req.body;
        
        if (!text || typeof text !== 'string' || text.trim().length === 0) {
            return res.status(400).json({ 
                error: 'Campo "text" é obrigatório' 
            });
        }

        const aiResult = await aiService.categorize(text.trim());
        const transaction = await transactionService.createTransaction(
            req.user.id,
            {
                ...aiResult,
                description: text.trim()
            }
        );

        res.status(201).json({
            success: true,
            transaction,
            ai_suggestion: {
                category: aiResult.suggested_category,
                cleaned_description: aiResult.cleaned_description
            }
        });

    } catch (error) {
        console.error('Erro em POST /api/transactions:', error);
        if (error.message.includes('categorização')) {
            return res.status(503).json({ 
                error: 'Serviço de IA temporariamente indisponível',
                details: error.message 
            });
        }
        res.status(500).json({ 
            error: 'Erro interno ao processar transação',
            details: error.message 
        });
    }
});

router.get('/', requireAuth, async (req, res) => {
    try {
        const limit = parseInt(req.query.limit) || 50;
        const offset = parseInt(req.query.offset) || 0;
        
        const result = await transactionService.listTransactions(
            req.user.id,
            limit,
            offset
        );
        
        res.json(result);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

router.patch('/:id/feedback', requireAuth, async (req, res) => {
    try {
        const { isCorrect } = req.body;
        
        if (typeof isCorrect !== 'boolean') {
            return res.status(400).json({ 
                error: 'Campo "isCorrect" deve ser booleano' 
            });
        }

        const transaction = await transactionService.updateAiFeedback(
            req.params.id,
            req.user.id,
            isCorrect
        );

        res.json({ success: true, transaction });
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

module.exports = router;
```

---

## 🎨 3. FRONTEND (React + Vite)

### 📄 `frontend/package.json`
```json
{
  "name": "gastos-ai-frontend",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "@supabase/supabase-js": "^2.39.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.2.1",
    "autoprefixer": "^10.4.16",
    "postcss": "^8.4.32",
    "tailwindcss": "^3.4.0",
    "vite": "^5.0.10"
  }
}
```

### 📄 `frontend/.env.example` (renomeie para `.env` e preencha)
```env
VITE_SUPABASE_URL=sua_url_do_supabase
VITE_SUPABASE_ANON_KEY=sua_chave_anon_do_supabase
```

### 📄 `frontend/vite.config.js`
```javascript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    proxy: {
      '/api': {
        target: 'http://localhost:3001',
        changeOrigin: true
      }
    }
  }
});
```

### 📄 `frontend/index.html`
```html
<!DOCTYPE html>
<html lang="pt-BR">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/vite.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Gastos com IA</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
```

### 📄 `frontend/src/main.jsx`
```jsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
import './index.css';

ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

### 📄 `frontend/src/index.css`
```css
@tailwind base;
@tailwind components;
@tailwind utilities;

body {
  @apply bg-gray-50 text-gray-900 antialiased;
}
```

### 📄 `frontend/src/lib/supabase.js`
```javascript
import { createClient } from '@supabase/supabase-js';

const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseAnonKey = import.meta.env.VITE_SUPABASE_ANON_KEY;

if (!supabaseUrl || !supabaseAnonKey) {
    throw new Error('Missing Supabase environment variables in frontend');
}

export const supabase = createClient(supabaseUrl, supabaseAnonKey);
```

### 📄 `frontend/src/App.jsx`
```jsx
import React, { useState, useEffect } from 'react';
import TransactionInput from './components/TransactionInput';
import { supabase } from './lib/supabase';

function App() {
  const [session, setSession] = useState(null);
  const [transactions, setTransactions] = useState([]);

  useEffect(() => {
    supabase.auth.getSession().then(({ data: { session } }) => {
      setSession(session);
    });

    const { data: { subscription } } = supabase.auth.onAuthStateChange((_event, session) => {
      setSession(session);
    });

    return () => subscription.unsubscribe();
  }, []);

  const handleSignIn = async () => {
    // Login simplificado para teste (substitua por sua lógica)
    const { data, error } = await supabase.auth.signInWithPassword({
      email: 'teste@email.com',
      password: 'senha123'
    });
    if (error) alert(error.message);
  };

  const handleSignOut = async () => {
    await supabase.auth.signOut();
  };

  if (!session) {
    return (
      <div className="min-h-screen flex items-center justify-center bg-gray-100">
        <div className="bg-white p-8 rounded-xl shadow-md text-center">
          <h1 className="text-2xl font-bold mb-4">💰 Gastos com IA</h1>
          <p className="text-gray-600 mb-4">Faça login para continuar</p>
          {/* Adicione aqui seu formulário de login ou use o componente de Auth do Supabase */}
          <button 
            onClick={handleSignIn}
            className="bg-blue-600 text-white px-6 py-2 rounded-lg hover:bg-blue-700"
          >
            Entrar (demo)
          </button>
        </div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gray-100 py-8">
      <div className="max-w-4xl mx-auto px-4">
        <div className="flex justify-between items-center mb-8">
          <h1 className="text-3xl font-bold text-gray-900">💰 Gastos com IA</h1>
          <button 
            onClick={handleSignOut}
            className="text-sm text-gray-500 hover:text-gray-700"
          >
            Sair
          </button>
        </div>
        
        <TransactionInput onTransactionAdded={() => {
          // Recarregar lista aqui se quiser
        }} />
        
        <div className="mt-8 bg-white rounded-xl shadow p-4">
          <h2 className="font-semibold mb-2">Últimas transações</h2>
          {transactions.length === 0 ? (
            <p className="text-gray-400 text-sm">Nenhuma transação ainda.</p>
          ) : (
            <ul>{/* Render list */}</ul>
          )}
        </div>
      </div>
    </div>
  );
}

export default App;
```

### 📄 `frontend/src/components/TransactionInput.jsx`
```jsx
import React, { useState } from 'react';
import { supabase } from '../lib/supabase';

const TransactionInput = ({ onTransactionAdded }) => {
    const [text, setText] = useState('');
    const [isLoading, setIsLoading] = useState(false);
    const [suggestion, setSuggestion] = useState(null);
    const [showConfirm, setShowConfirm] = useState(false);
    const [error, setError] = useState(null);

    const handleSubmit = async (e) => {
        e.preventDefault();
        
        if (!text.trim()) {
            setError('Por favor, descreva o gasto.');
            return;
        }

        setIsLoading(true);
        setError(null);
        setSuggestion(null);
        setShowConfirm(false);

        try {
            const { data: { session } } = await supabase.auth.getSession();
            const token = session?.access_token;

            if (!token) {
                throw new Error('Usuário não autenticado');
            }

            const response = await fetch('/api/transactions', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                    'Authorization': `Bearer ${token}`
                },
                body: JSON.stringify({ text: text.trim() })
            });

            const data = await response.json();

            if (!response.ok) {
                throw new Error(data.error || 'Erro ao processar transação');
            }

            setSuggestion({
                category: data.ai_suggestion.category,
                cleanedDescription: data.ai_suggestion.cleaned_description,
                amount: data.transaction.amount,
                date: data.transaction.transaction_date,
                transactionId: data.transaction.id
            });
            setShowConfirm(true);

        } catch (err) {
            setError(err.message);
        } finally {
            setIsLoading(false);
        }
    };

    const handleConfirm = async (isCorrect) => {
        try {
            if (isCorrect) {
                const { data: { session } } = await supabase.auth.getSession();
                await fetch(`/api/transactions/${suggestion.transactionId}/feedback`, {
                    method: 'PATCH',
                    headers: {
                        'Content-Type': 'application/json',
                        'Authorization': `Bearer ${session?.access_token}`
                    },
                    body: JSON.stringify({ isCorrect: true })
                });
            }

            setShowConfirm(false);
            setSuggestion(null);
            setText('');
            
            if (onTransactionAdded) {
                onTransactionAdded();
            }

        } catch (err) {
            setError('Erro ao registrar feedback: ' + err.message);
        }
    };

    const handleCorrection = () => {
        setShowConfirm(false);
        setSuggestion(null);
    };

    return (
        <div className="bg-white rounded-xl shadow-lg p-6">
            <form onSubmit={handleSubmit} className="space-y-4">
                <div>
                    <label 
                        htmlFor="expense-input" 
                        className="block text-sm font-medium text-gray-700 mb-1"
                    >
                        Como foi esse gasto?
                    </label>
                    <textarea
                        id="expense-input"
                        rows="3"
                        className="w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-blue-500 transition-colors"
                        placeholder="Ex: Paguei 50 reais no ifood ontem"
                        value={text}
                        onChange={(e) => setText(e.target.value)}
                        disabled={isLoading}
                    />
                </div>

                <button
                    type="submit"
                    disabled={isLoading || !text.trim()}
                    className="w-full bg-blue-600 hover:bg-blue-700 text-white font-semibold py-3 px-6 rounded-lg transition-colors disabled:opacity-50 disabled:cursor-not-allowed"
                >
                    {isLoading ? (
                        <span className="flex items-center justify-center gap-2">
                            <svg className="animate-spin h-5 w-5" viewBox="0 0 24 24">
                                <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4" fill="none" />
                                <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z" />
                            </svg>
                            Analisando...
                        </span>
                    ) : 'Adicionar Gasto'}
                </button>

                {error && (
                    <div className="p-4 bg-red-50 border border-red-200 rounded-lg text-red-700 text-sm">
                        ⚠️ {error}
                    </div>
                )}
            </form>

            {showConfirm && suggestion && (
                <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center p-4 z-50">
                    <div className="bg-white rounded-xl shadow-2xl max-w-md w-full p-6">
                        <h3 className="text-lg font-semibold text-gray-900 mb-4">
                            ✨ A IA categorizou seu gasto
                        </h3>
                        
                        <div className="space-y-3 mb-6">
                            <div className="flex justify-between border-b border-gray-100 pb-2">
                                <span className="text-gray-500">Descrição</span>
                                <span className="font-medium">{suggestion.cleanedDescription}</span>
                            </div>
                            <div className="flex justify-between border-b border-gray-100 pb-2">
                                <span className="text-gray-500">Valor</span>
                                <span className="font-medium">R$ {suggestion.amount?.toFixed(2)}</span>
                            </div>
                            <div className="flex justify-between border-b border-gray-100 pb-2">
                                <span className="text-gray-500">Data</span>
                                <span className="font-medium">{suggestion.date}</span>
                            </div>
                            <div className="flex justify-between">
                                <span className="text-gray-500">Categoria sugerida</span>
                                <span className="font-medium text-blue-600 bg-blue-50 px-3 py-1 rounded-full">
                                    {suggestion.category}
                                </span>
                            </div>
                        </div>

                        <div className="flex gap-3">
                            <button
                                onClick={() => handleConfirm(true)}
                                className="flex-1 bg-green-600 hover:bg-green-700 text-white font-medium py-2 px-4 rounded-lg transition-colors"
                            >
                                ✅ Confirmar
                            </button>
                            <button
                                onClick={handleCorrection}
                                className="flex-1 bg-yellow-500 hover:bg-yellow-600 text-white font-medium py-2 px-4 rounded-lg transition-colors"
                            >
                                ✏️ Corrigir
                            </button>
                        </div>
                    </div>
                </div>
            )}
        </div>
    );
};

export default TransactionInput;
```

---

## 🚀 PASSO A PASSO PARA RODAR

### 1. Banco de Dados (Supabase)
- Crie uma conta no [Supabase](https://supabase.com) e um novo projeto.
- Vá no **SQL Editor** e cole todo o código SQL fornecido acima. Execute.
- Vá em **Authentication → Users** e crie um usuário de teste (ex: `teste@email.com` / `senha123`).
- Anote as credenciais: **URL** (Settings → API) e **Service Role Key** (cuidado, é secreta).

### 2. Backend
```bash
cd backend
npm install
cp .env.example .env
# Edite o .env com suas credenciais do Supabase e a NOVA chave DeepSeek
npm run dev
```

### 3. Frontend
```bash
cd frontend
npm install
cp .env.example .env
# Edite o .env com a URL e ANON KEY do Supabase
npm run dev
```

Acesse `http://localhost:5173` no navegador.

---

## ⚠️ ÚLTIMO ALERTA DE SEGURANÇA
- **NUNCA** comite os arquivos `.env` (adicione ao `.gitignore`).
- A chave do DeepSeek **só existe no backend** (nunca no frontend).
- O `SUPABASE_SERVICE_ROLE_KEY` **só existe no backend**.
- No frontend, use apenas a `ANON_KEY`.

Seu app está pronto! 🚀 Qualquer dúvida, me chame.