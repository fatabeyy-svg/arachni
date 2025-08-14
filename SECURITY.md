// ===== PROJE YAPISI =====
// call-panel/
// ├── package.json
// ├── .env.local
// ├── prisma/
// │   └── schema.prisma
// ├── pages/
// │   ├── api/
// │   │   ├── auth/
// │   │   │   ├── login.js
// │   │   │   └── logout.js
// │   │   ├── users/
// │   │   │   └── [...].js
// │   │   ├── members/
// │   │   │   └── [...].js
// │   │   ├── calls/
// │   │   │   └── [...].js
// │   │   └── notifications/
// │   │       └── [...].js
// │   ├── index.js
// │   ├── dashboard.js
// │   └── admin.js
// ├── components/
// │   ├── Layout.js
// │   ├── Sidebar.js
// │   └── [...].js
// └── lib/
//     ├── prisma.js
//     └── auth.js

// ===== 1. PACKAGE.JSON =====
{
  "name": "call-panel",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "db:push": "prisma db push",
    "db:seed": "node prisma/seed.js"
  },
  "dependencies": {
    "next": "14.0.0",
    "react": "18.2.0",
    "react-dom": "18.2.0",
    "@prisma/client": "5.7.0",
    "prisma": "5.7.0",
    "bcryptjs": "2.4.3",
    "jsonwebtoken": "9.0.2",
    "cookie": "0.6.0",
    "axios": "1.6.2",
    "react-hot-toast": "2.4.1",
    "react-icons": "4.12.0",
    "date-fns": "2.30.0",
    "recharts": "2.10.3",
    "xlsx": "0.18.5",
    "node-cron": "3.0.3"
  }
}

// ===== 2. .ENV.LOCAL =====
DATABASE_URL="postgresql://username:password@localhost:5432/callpanel"
JWT_SECRET="your-super-secret-jwt-key-change-this"
NEXTAUTH_URL="http://localhost:3000"

// ===== 3. PRISMA SCHEMA (prisma/schema.prisma) =====
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

enum Role {
  ADMIN
  SUPERVISOR
  OPERATOR
}

enum MemberStatus {
  ACTIVE
  PASSIVE
  INVESTED
  LEFT_INVESTMENT
  PENDING_CHECK
}

enum CallStatus {
  REACHED
  BUSY
  NO_ANSWER
  APPOINTMENT
  INVESTED
  CHECK_REQUIRED
}

model User {
  id        String   @id @default(cuid())
  username  String   @unique
  password  String
  name      String
  role      Role     @default(OPERATOR)
  active    Boolean  @default(true)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  calls         Call[]
  notifications Notification[]
  activities    Activity[]
}

model Member {
  id            String       @id @default(cuid())
  username      String       @unique
  status        MemberStatus @default(ACTIVE)
  tags          String[]     @default([])
  note          String?
  investmentDate DateTime?
  checkDate     DateTime?
  leftDate      DateTime?
  createdAt     DateTime     @default(now())
  updatedAt     DateTime     @updatedAt
  
  calls         Call[]
  notifications Notification[]
}

model Call {
  id            String     @id @default(cuid())
  memberId      String
  userId        String
  status        CallStatus
  duration      Int?
  note          String
  nextCallDate  DateTime?
  investmentAmount Float?
  createdAt     DateTime   @default(now())
  
  member Member @relation(fields: [memberId], references: [id])
  user   User   @relation(fields: [userId], references: [id])
}

model Notification {
  id        String   @id @default(cuid())
  userId    String?
  memberId  String
  type      String   // INVESTMENT_CHECK, FOLLOW_UP, etc.
  message   String
  read      Boolean  @default(false)
  dueDate   DateTime
  createdAt DateTime @default(now())
  
  user   User?  @relation(fields: [userId], references: [id])
  member Member @relation(fields: [memberId], references: [id])
}

model Activity {
  id        String   @id @default(cuid())
  userId    String
  action    String
  details   String?
  createdAt DateTime @default(now())
  
  user User @relation(fields: [userId], references: [id])
}

model Tag {
  id    String @id @default(cuid())
  name  String @unique
  color String
}

// ===== 4. DATABASE SEED (prisma/seed.js) =====
const { PrismaClient } = require('@prisma/client');
const bcrypt = require('bcryptjs');

const prisma = new PrismaClient();

async function main() {
  // Create admin user
  const adminPassword = await bcrypt.hash('admin123', 10);
  await prisma.user.create({
    data: {
      username: 'admin',
      password: adminPassword,
      name: 'Admin User',
      role: 'ADMIN'
    }
  });

  // Create supervisor
  const supervisorPassword = await bcrypt.hash('super123', 10);
  await prisma.user.create({
    data: {
      username: 'supervisor',
      password: supervisorPassword,
      name: 'Supervisor User',
      role: 'SUPERVISOR'
    }
  });

  // Create operator
  const operatorPassword = await bcrypt.hash('op123', 10);
  await prisma.user.create({
    data: {
      username: 'operator1',
      password: operatorPassword,
      name: 'Operator 1',
      role: 'OPERATOR'
    }
  });

  // Create default tags
  const tags = [
    { name: 'VIP', color: '#FFD700' },
    { name: 'Yeni Üye', color: '#10B981' },
    { name: 'Risk', color: '#EF4444' },
    { name: 'Potansiyel', color: '#3B82F6' },
    { name: 'Yatırım Yaptı', color: '#8B5CF6' },
    { name: 'Yatırım Bıraktı', color: '#F59E0B' },
    { name: 'Kontrol Bekliyor', color: '#06B6D4' }
  ];

  for (const tag of tags) {
    await prisma.tag.create({ data: tag });
  }

  console.log('Database seeded successfully!');
}

main()
  .catch((e) => console.error(e))
  .finally(async () => await prisma.$disconnect());

// ===== 5. PRISMA CLIENT (lib/prisma.js) =====
import { PrismaClient } from '@prisma/client';

let prisma;

if (process.env.NODE_ENV === 'production') {
  prisma = new PrismaClient();
} else {
  if (!global.prisma) {
    global.prisma = new PrismaClient();
  }
  prisma = global.prisma;
}

export default prisma;

// ===== 6. AUTH MIDDLEWARE (lib/auth.js) =====
import jwt from 'jsonwebtoken';
import cookie from 'cookie';

export function verifyToken(req) {
  const cookies = cookie.parse(req.headers.cookie || '');
  const token = cookies.token;
  
  if (!token) {
    return null;
  }
  
  try {
    return jwt.verify(token, process.env.JWT_SECRET);
  } catch (error) {
    return null;
  }
}

export function requireAuth(handler, roles = []) {
  return async (req, res) => {
    const user = verifyToken(req);
    
    if (!user) {
      return res.status(401).json({ error: 'Unauthorized' });
    }
    
    if (roles.length > 0 && !roles.includes(user.role)) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    
    req.user = user;
    return handler(req, res);
  };
}

// ===== 7. LOGIN API (pages/api/auth/login.js) =====
import bcrypt from 'bcryptjs';
import jwt from 'jsonwebtoken';
import cookie from 'cookie';
import prisma from '../../../lib/prisma';

export default async function handler(req, res) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed' });
  }
  
  const { username, password } = req.body;
  
  try {
    const user = await prisma.user.findUnique({
      where: { username }
    });
    
    if (!user || !await bcrypt.compare(password, user.password)) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    if (!user.active) {
      return res.status(403).json({ error: 'Account deactivated' });
    }
    
    const token = jwt.sign(
      { 
        id: user.id, 
        username: user.username, 
        role: user.role,
        name: user.name 
      },
      process.env.JWT_SECRET,
      { expiresIn: '8h' }
    );
    
    res.setHeader(
      'Set-Cookie',
      cookie.serialize('token', token, {
        httpOnly: true,
        secure: process.env.NODE_ENV === 'production',
        sameSite: 'strict',
        maxAge: 28800, // 8 hours
        path: '/'
      })
    );
    
    // Log activity
    await prisma.activity.create({
      data: {
        userId: user.id,
        action: 'LOGIN',
        details: `${user.name} giriş yaptı`
      }
    });
    
    res.status(200).json({ 
      user: {
        id: user.id,
        username: user.username,
        name: user.name,
        role: user.role
      }
    });
  } catch (error) {
    console.error('Login error:', error);
    res.status(500).json({ error: 'Server error' });
  }
}

// ===== 8. MEMBERS API (pages/api/members/index.js) =====
import prisma from '../../../lib/prisma';
import { requireAuth } from '../../../lib/auth';

async function handler(req, res) {
  if (req.method === 'GET') {
    // Get all members with filters
    const { status, tag, search } = req.query;
    
    const where = {};
    if (status) where.status = status;
    if (tag) where.tags = { has: tag };
    if (search) {
      where.OR = [
        { username: { contains: search, mode: 'insensitive' } },
        { note: { contains: search, mode: 'insensitive' } }
      ];
    }
    
    const members = await prisma.member.findMany({
      where,
      include: {
        calls: {
          orderBy: { createdAt: 'desc' },
          take: 1
        }
      },
      orderBy: { createdAt: 'desc' }
    });
    
    return res.status(200).json(members);
  }
  
  if (req.method === 'POST') {
    const { username, status, tags, note } = req.body;
    
    try {
      const member = await prisma.member.create({
        data: {
          username,
          status: status || 'ACTIVE',
          tags: tags || [],
          note
        }
      });
      
      // Log activity
      await prisma.activity.create({
        data: {
          userId: req.user.id,
          action: 'MEMBER_CREATED',
          details: `${req.user.name} yeni üye ekledi: ${username}`
        }
      });
      
      return res.status(201).json(member);
    } catch (error) {
      if (error.code === 'P2002') {
        return res.status(400).json({ error: 'Username already exists' });
      }
      throw error;
    }
  }
  
  return res.status(405).json({ error: 'Method not allowed' });
}

export default requireAuth(handler);

// ===== 9. CALLS API (pages/api/calls/index.js) =====
import prisma from '../../../lib/prisma';
import { requireAuth } from '../../../lib/auth';
import { addDays } from 'date-fns';

async function handler(req, res) {
  if (req.method === 'GET') {
    const { memberId, userId, status, from, to } = req.query;
    
    const where = {};
    if (memberId) where.memberId = memberId;
    if (userId) where.userId = userId;
    if (status) where.status = status;
    if (from || to) {
      where.createdAt = {};
      if (from) where.createdAt.gte = new Date(from);
      if (to) where.createdAt.lte = new Date(to);
    }
    
    const calls = await prisma.call.findMany({
      where,
      include: {
        member: true,
        user: {
          select: {
            id: true,
            name: true,
            username: true
          }
        }
      },
      orderBy: { createdAt: 'desc' }
    });
    
    return res.status(200).json(calls);
  }
  
  if (req.method === 'POST') {
    const { 
      memberId, 
      status, 
      duration, 
      note, 
      nextCallDate,
      investmentAmount 
    } = req.body;
    
    try {
      // Create call record
      const call = await prisma.call.create({
        data: {
          memberId,
          userId: req.user.id,
          status,
          duration,
          note,
          nextCallDate: nextCallDate ? new Date(nextCallDate) : null,
          investmentAmount
        },
        include: {
          member: true,
          user: true
        }
      });
      
      // Update member status if invested
      if (status === 'INVESTED') {
        await prisma.member.update({
          where: { id: memberId },
          data: {
            status: 'INVESTED',
            investmentDate: new Date(),
            checkDate: addDays(new Date(), 3),
            tags: {
              push: 'Yatırım Yaptı'
            }
          }
        });
        
        // Create notification for 3 days later
        await prisma.notification.create({
          data: {
            memberId,
            type: 'INVESTMENT_CHECK',
            message: `${call.member.username} yatırım kontrolü yapılmalı (3 gün geçti)`,
            dueDate: addDays(new Date(), 3)
          }
        });
      }
      
      // Log activity
      await prisma.activity.create({
        data: {
          userId: req.user.id,
          action: 'CALL_CREATED',
          details: `${req.user.name} arama kaydı oluşturdu: ${call.member.username}`
        }
      });
      
      return res.status(201).json(call);
    } catch (error) {
      console.error('Call creation error:', error);
      return res.status(500).json({ error: 'Failed to create call' });
    }
  }
  
  return res.status(405).json({ error: 'Method not allowed' });
}

export default requireAuth(handler);

// ===== 10. NOTIFICATIONS CHECK (pages/api/notifications/check.js) =====
import prisma from '../../../lib/prisma';
import { requireAuth } from '../../../lib/auth';

async function handler(req, res) {
  if (req.method === 'GET') {
    const notifications = await prisma.notification.findMany({
      where: {
        read: false,
        dueDate: {
          lte: new Date()
        }
      },
      include: {
        member: true
      },
      orderBy: { dueDate: 'asc' }
    });
    
    return res.status(200).json(notifications);
  }
  
  if (req.method === 'PUT') {
    const { notificationId, action } = req.body;
    
    if (action === 'mark-read') {
      await prisma.notification.update({
        where: { id: notificationId },
        data: { read: true }
      });
    }
    
    return res.status(200).json({ success: true });
  }
  
  return res.status(405).json({ error: 'Method not allowed' });
}

export default requireAuth(handler);

// ===== 11. INVESTMENT CHECK API (pages/api/members/check-investment.js) =====
import prisma from '../../../lib/prisma';
import { requireAuth } from '../../../lib/auth';

async function handler(req, res) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed' });
  }
  
  const { memberId, stillInvested, note } = req.body;
  
  try {
    const member = await prisma.member.findUnique({
      where: { id: memberId }
    });
    
    if (!member) {
      return res.status(404).json({ error: 'Member not found' });
    }
    
    if (stillInvested) {
      // Still invested, schedule next check
      await prisma.member.update({
        where: { id: memberId },
        data: {
          checkDate: addDays(new Date(), 7), // Check again in 7 days
        }
      });
      
      await prisma.notification.create({
        data: {
          memberId,
          type: 'INVESTMENT_CHECK',
          message: `${member.username} yatırım kontrolü (haftalık)`,
          dueDate: addDays(new Date(), 7)
        }
      });
      
      // Create call record
      await prisma.call.create({
        data: {
          memberId,
          userId: req.user.id,
          status: 'CHECK_REQUIRED',
          note: `Yatırım kontrolü yapıldı. ${note}`
        }
      });
    } else {
      // Left investment
      await prisma.member.update({
        where: { id: memberId },
        data: {
          status: 'LEFT_INVESTMENT',
          leftDate: new Date(),
          tags: {
            push: 'Yatırım Bıraktı'
          }
        }
      });
      
      // Create call record
      await prisma.call.create({
        data: {
          memberId,
          userId: req.user.id,
          status: 'REACHED',
          note: `Yatırım bıraktı. ${note}`
        }
      });
    }
    
    // Log activity
    await prisma.activity.create({
      data: {
        userId: req.user.id,
        action: stillInvested ? 'INVESTMENT_CONTINUED' : 'INVESTMENT_LEFT',
        details: `${req.user.name} yatırım kontrolü yaptı: ${member.username}`
      }
    });
    
    return res.status(200).json({ success: true });
  } catch (error) {
    console.error('Investment check error:', error);
    return res.status(500).json({ error: 'Failed to check investment' });
  }
}

export default requireAuth(handler, ['SUPERVISOR', 'ADMIN']);

// ===== 12. DASHBOARD PAGE (pages/dashboard.js) =====
import { useState, useEffect } from 'react';
import { useRouter } from 'next/router';
import axios from 'axios';
import toast, { Toaster } from 'react-hot-toast';
import { FiPhone, FiUsers, FiTrendingUp, FiBell, FiLogOut, FiPlus, FiSearch, FiFilter } from 'react-icons/fi';

export default function Dashboard() {
  const router = useRouter();
  const [user, setUser] = useState(null);
  const [members, setMembers] = useState([]);
  const [calls, setCalls] = useState([]);
  const [notifications, setNotifications] = useState([]);
  const [stats, setStats] = useState({
    totalMembers: 0,
    todayCalls: 0,
    pendingChecks: 0,
    investmentCount: 0
  });
  const [activeTab, setActiveTab] = useState('members');
  const [showAddMember, setShowAddMember] = useState(false);
  const [showAddCall, setShowAddCall] = useState(false);
  const [selectedMember, setSelectedMember] = useState(null);
  const [filters, setFilters] = useState({
    status: '',
    tag: '',
    search: ''
  });

  useEffect(() => {
    // Check authentication
    const token = document.cookie.includes('token=');
    if (!token) {
      router.push('/');
      return;
    }
    
    // Get user info from localStorage or make API call
    const userData = localStorage.getItem('user');
    if (userData) {
      setUser(JSON.parse(userData));
    }
    
    loadData();
    checkNotifications();
    
    // Set up notification check interval
    const interval = setInterval(checkNotifications, 60000); // Check every minute
    return () => clearInterval(interval);
  }, []);

  const loadData = async () => {
    try {
      // Load members
      const membersRes = await axios.get('/api/members', { params: filters });
      setMembers(membersRes.data);
      
      // Load calls
      const callsRes = await axios.get('/api/calls');
      setCalls(callsRes.data);
      
      // Calculate stats
      const today = new Date().toDateString();
      const todayCallsCount = callsRes.data.filter(
        call => new Date(call.createdAt).toDateString() === today
      ).length;
      
      const investedMembers = membersRes.data.filter(
        m => m.status === 'INVESTED'
      ).length;
      
      const pendingChecks = membersRes.data.filter(
        m => m.status === 'PENDING_CHECK' || 
        (m.checkDate && new Date(m.checkDate) <= new Date())
      ).length;
      
      setStats({
        totalMembers: membersRes.data.length,
        todayCalls: todayCallsCount,
        pendingChecks,
        investmentCount: investedMembers
      });
    } catch (error) {
      toast.error('Veri yüklenirken hata oluştu');
      console.error('Load data error:', error);
    }
  };

  const checkNotifications = async () => {
    try {
      const res = await axios.get('/api/notifications/check');
      setNotifications(res.data);
      
      // Show toast for new notifications
      if (res.data.length > 0) {
        res.data.forEach(notif => {
          if (!notif.read) {
            toast(
              <div>
                <strong>Hatırlatma!</strong>
                <p>{notif.message}</p>
                <button 
                  onClick={() => handleNotification(notif)}
                  className="btn-small"
                >
                  İncele
                </button>
              </div>,
              { duration: 10000, icon: '🔔' }
            );
          }
        });
      }
    } catch (error) {
      console.error('Notification check error:', error);
    }
  };

  const handleNotification = async (notification) => {
    // Mark as read
    await axios.put('/api/notifications/check', {
      notificationId: notification.id,
      action: 'mark-read'
    });
    
    // Open member details or call modal
    setSelectedMember(notification.member);
    setShowAddCall(true);
    
    // Reload notifications
    checkNotifications();
  };

  const handleAddMember = async (data) => {
    try {
      await axios.post('/api/members', data);
      toast.success('Üye başarıyla eklendi');
      setShowAddMember(false);
      loadData();
    } catch (error) {
      toast.error(error.response?.data?.error || 'Üye eklenirken hata oluştu');
    }
  };

  const handleAddCall = async (data) => {
    try {
      await axios.post('/api/calls', {
        ...data,
        memberId: selectedMember?.id || data.memberId
      });
      
      toast.success('Arama kaydı eklendi');
      
      // Check if investment made
      if (data.status === 'INVESTED') {
        toast.success('3 gün sonra kontrol hatırlatması oluşturuldu', {
          icon: '📅',
          duration: 5000
        });
      }
      
      setShowAddCall(false);
      setSelectedMember(null);
      loadData();
    } catch (error) {
      toast.error('Arama kaydı eklenirken hata oluştu');
    }
  };

  const handleInvestmentCheck = async (memberId, stillInvested, note) => {
    try {
      await axios.post('/api/members/check-investment', {
        memberId,
        stillInvested,
        note
      });
      
      toast.success(
        stillInvested 
          ? 'Yatırım devam ediyor, 7 gün sonra tekrar kontrol edilecek'
          : 'Üye yatırım bıraktı olarak işaretlendi'
      );
      
      loadData();
    } catch (error) {
      toast.error('Kontrol güncellenirken hata oluştu');
    }
  };

  const logout = () => {
    document.cookie = 'token=; Max-Age=0; path=/';
    localStorage.removeItem('user');
    router.push('/');
  };

  return (
    <div className="dashboard">
      <Toaster position="top-right" />
      
      {/* Header */}
      <header className="header">
        <div className="header-content">
          <h1>📞 Call Panel</h1>
          <div className="header-right">
            <div className="notifications">
              <FiBell />
              {notifications.length > 0 && (
                <span className="badge">{notifications.length}</span>
              )}
            </div>
            <div className="user-info">
              <span>{user?.name}</span>
              <span className="role-badge">{user?.role}</span>
            </div>
            <button onClick={logout} className="btn-icon">
              <FiLogOut />
            </button>
          </div>
        </div>
      </header>

      {/* Stats */}
      <div className="stats-grid">
        <div className="stat-card">
          <div className="stat-icon">
            <FiUsers />
          </div>
          <div className="stat-content">
            <div className="stat-title">Toplam Üye</div>
            <div className="stat-value">{stats.totalMembers}</div>
          </div>
        </div>
        
        <div className="stat-card success">
          <div className="stat-icon">
            <FiPhone />
          </div>
          <div className="stat-content">
            <div className="stat-title">Bugünkü Aramalar</div>
            <div className="stat-value">{stats.todayCalls}</div>
          </div>
        </div>
        
        <div className="stat-card warning">
          <div className="stat-icon">
            <FiBell />
          </div>
          <div className="stat-content">
            <div className="stat-title">Kontrol Bekleyen</div>
            <div className="stat-value">{stats.pendingChecks}</div>
          </div>
        </div>
        
        <div className="stat-card primary">
          <div className="stat-icon">
            <FiTrendingUp />
          </div>
          <div className="stat-content">
            <div className="stat-title">Yatırım Yapan</div>
            <div className="stat-value">{stats.investmentCount}</div>
          </div>
        </div>
      </div>

      {/* Tabs */}
      <div className="tabs">
        <button 
          className={activeTab === 'members' ? 'active' : ''}
          onClick={() => setActiveTab('members')}
        >
          Üyeler
        </button>
        <button 
          className={activeTab === 'calls' ? 'active' : ''}
          onClick={() => setActiveTab('calls')}
        >
          Aramalar
        </button>
        <button 
          className={activeTab === 'reports' ? 'active' : ''}
          onClick={() => setActiveTab('reports')}
        >
          Raporlar
        </button>
        {(user?.role === 'ADMIN' || user?.role === 'SUPERVISOR') && (
          <button 
            className={activeTab === 'admin' ? 'active' : ''}
            onClick={() => setActiveTab('admin')}
          >
            Yönetim
          </button>
        )}
      </div>

      {/* Content */}
      <div className="content">
        {activeTab === 'members' && (
          <div className="members-section">
            <div className="section-header">
              <h2>Üye Listesi</h2>
              <button 
                className="btn btn-primary"
                onClick={() => setShowAddMember(true)}
              >
                <FiPlus /> Yeni Üye
              </button>
            </div>
            
            {/* Filters */}
            <div className="filters">
              <div className="search-box">
                <FiSearch />
                <input 
                  type="text"
                  placeholder="Üye ara..."
                  value={filters.search}
                  onChange={(e) => setFilters({...filters, search: e.target.value})}
                />
              </div>
              
              <select 
                value={filters.status}
                onChange={(e) => setFilters({...filters, status: e.target.value})}
              >
                <option value="">Tüm Durumlar</option>
                <option value="ACTIVE">Aktif</option>
                <option value="PASSIVE">Pasif</option>
                <option value="INVESTED">Yatırım Yaptı</option>
                <option value="LEFT_INVESTMENT">Yatırım Bıraktı</option>
                <option value="PENDING_CHECK">Kontrol Bekliyor</option>
              </select>
              
              <button 
                className="btn btn-secondary"
                onClick={loadData}
              >
                <FiFilter /> Filtrele
              </button>
            </div>
            
            {/* Members Table */}
            <table className="data-table">
              <thead>
                <tr>
                  <th>Kullanıcı Adı</th>
                  <th>Durum</th>
                  <th>Etiketler</th>
                  <th>Son Arama</th>
                  <th>Yatırım Durumu</th>
                  <th>İşlemler</th>
                </tr>
              </thead>
              <tbody>
                {members.map(member => (
                  <tr key={member.id}>
                    <td><strong>{member.username}</strong></td>
                    <td>
                      <span className={`status-badge ${member.status.toLowerCase()}`}>
                        {member.status === 'ACTIVE' && 'Aktif'}
                        {member.status === 'PASSIVE' && 'Pasif'}
                        {member.status === 'INVESTED' && 'Yatırım Yaptı'}
                        {member.status === 'LEFT_INVESTMENT' && 'Yatırım Bıraktı'}
                        {member.status === 'PENDING_CHECK' && 'Kontrol Bekliyor'}
                      </span>
                    </td>
                    <td>
                      <div className="tags">
                        {member.tags?.map(tag => (
                          <span key={tag} className="tag">{tag}</span>
                        ))}
                      </div>
                    </td>
                    <td>
                      {member.calls?.[0] 
                        ? new Date(member.calls[0].createdAt).toLocaleDateString('tr-TR')
                        : 'Henüz aranmadı'
                      }
                    </td>
                    <td>
                      {member.investmentDate && (
                        <div>
                          <small>Yatırım: {new Date(member.investmentDate).toLocaleDateString('tr-TR')}</small>
                          {member.checkDate && new Date(member.checkDate) <= new Date() && (
                            <span className="alert-badge">Kontrol gerekli!</span>
                          )}
                        </div>
                      )}
                    </td>
                    <td>
                      <div className="action-buttons">
                        <button 
                          className="btn-icon"
                          onClick={() => {
                            setSelectedMember(member);
                            setShowAddCall(true);
                          }}
                        >
                          <FiPhone />
                        </button>
                        
                        {member.status === 'INVESTED' && member.checkDate && 
                         new Date(member.checkDate) <= new Date() && 
                         (user?.role === 'SUPERVISOR' || user?.role === 'ADMIN') && (
                          <button 
                            className="btn btn-warning btn-small"
                            onClick={() => {
                              const stillInvested = confirm('Üye hala yatırımda mı?');
                              const note = prompt('Not ekleyin:');
                              handleInvestmentCheck(member.id, stillInvested, note);
                            }}
                          >
                            Kontrol Et
                          </button>
                        )}
                      </div>
                    </td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        )}
        
        {/* Add more tab contents here... */}
      </div>

      {/* Modals */}
      {showAddMember && (
        <div className="modal">
          <div className="modal-content">
            <h3>Yeni Üye Ekle</h3>
            <form onSubmit={(e) => {
              e.preventDefault();
              const formData = new FormData(e.target);
              handleAddMember({
                username: formData.get('username'),
                status: formData.get('status'),
                note: formData.get('note'),
                tags: formData.get('tags')?.split(',').map(t => t.trim()) || []
              });
            }}>
              <input name="username" placeholder="Kullanıcı adı" required />
              <select name="status">
                <option value="ACTIVE">Aktif</option>
                <option value="PASSIVE">Pasif</option>
              </select>
              <input name="tags" placeholder="Etiketler (virgülle ayırın)" />
              <textarea name="note" placeholder="Not"></textarea>
              <div className="modal-actions">
                <button type="submit" className="btn btn-primary">Ekle</button>
                <button type="button" onClick={() => setShowAddMember(false)}>İptal</button>
              </div>
            </form>
          </div>
        </div>
      )}

      {showAddCall && (
        <div className="modal">
          <div className="modal-content">
            <h3>Arama Kaydı - {selectedMember?.username}</h3>
            <form onSubmit={(e) => {
              e.preventDefault();
              const formData = new FormData(e.target);
              handleAddCall({
                status: formData.get('status'),
                duration: parseInt(formData.get('duration')) || 0,
                note: formData.get('note'),
                investmentAmount: parseFloat(formData.get('investmentAmount')) || null,
                nextCallDate: formData.get('nextCallDate') || null
              });
            }}>
              <select name="status" required>
                <option value="REACHED">Ulaşıldı</option>
                <option value="BUSY">Meşgul</option>
                <option value="NO_ANSWER">Cevap Yok</option>
                <option value="APPOINTMENT">Randevu</option>
                <option value="INVESTED">Yatırım Yaptı ✨</option>
              </select>
              <input name="duration" type="number" placeholder="Görüşme süresi (dk)" />
              <input name="investmentAmount" type="number" step="0.01" placeholder="Yatırım tutarı (opsiyonel)" />
              <textarea name="note" placeholder="Görüşme notu" required></textarea>
              <input name="nextCallDate" type="datetime-local" />
              <div className="modal-actions">
                <button type="submit" className="btn btn-primary">Kaydet</button>
                <button type="button" onClick={() => {
                  setShowAddCall(false);
                  setSelectedMember(null);
                }}>İptal</button>
              </div>
            </form>
          </div>
        </div>
      )}

      <style jsx>{`
        .dashboard {
          min-height: 100vh;
          background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
          padding: 20px;
        }

        .header {
          background: white;
          border-radius: 12px;
          padding: 20px;
          margin-bottom: 20px;
          box-shadow: 0 10px 30px rgba(0,0,0,0.1);
        }

        .header-content {
          display: flex;
          justify-content: space-between;
          align-items: center;
        }

        .header-right {
          display: flex;
          align-items: center;
          gap: 20px;
        }

        .notifications {
          position: relative;
          cursor: pointer;
        }

        .notifications .badge {
          position: absolute;
          top: -8px;
          right: -8px;
          background: #ef4444;
          color: white;
          border-radius: 50%;
          width: 20px;
          height: 20px;
          display: flex;
          align-items: center;
          justify-content: center;
          font-size: 12px;
        }

        .user-info {
          display: flex;
          align-items: center;
          gap: 10px;
        }

        .role-badge {
          background: #4f46e5;
          color: white;
          padding: 4px 8px;
          border-radius: 4px;
          font-size: 12px;
        }

        .stats-grid {
          display: grid;
          grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
          gap: 20px;
          margin-bottom: 20px;
        }

        .stat-card {
          background: white;
          border-radius: 12px;
          padding: 20px;
          display: flex;
          align-items: center;
          gap: 15px;
          box-shadow: 0 10px 30px rgba(0,0,0,0.1);
        }

        .stat-card.success { border-left: 4px solid #10b981; }
        .stat-card.warning { border-left: 4px solid #f59e0b; }
        .stat-card.primary { border-left: 4px solid #4f46e5; }

        .stat-icon {
          width: 50px;
          height: 50px;
          background: #f3f4f6;
          border-radius: 10px;
          display: flex;
          align-items: center;
          justify-content: center;
          font-size: 24px;
        }

        .stat-title {
          font-size: 14px;
          color: #6b7280;
        }

        .stat-value {
          font-size: 28px;
          font-weight: bold;
          color: #1f2937;
        }

        .tabs {
          display: flex;
          gap: 10px;
          background: white;
          padding: 10px;
          border-radius: 12px;
          margin-bottom: 20px;
        }

        .tabs button {
          padding: 10px 20px;
          border: none;
          background: transparent;
          border-radius: 8px;
          cursor: pointer;
          font-weight: 500;
          transition: all 0.3s;
        }

        .tabs button.active {
          background: #4f46e5;
          color: white;
        }

        .content {
          background: white;
          border-radius: 12px;
          padding: 30px;
          min-height: 400px;
        }

        .section-header {
          display: flex;
          justify-content: space-between;
          align-items: center;
          margin-bottom: 20px;
        }

        .filters {
          display: flex;
          gap: 15px;
          margin-bottom: 20px;
        }

        .search-box {
          position: relative;
          flex: 1;
        }

        .search-box input {
          width: 100%;
          padding: 10px 10px 10px 40px;
          border: 2px solid #e5e7eb;
          border-radius: 8px;
        }

        .search-box svg {
          position: absolute;
          left: 12px;
          top: 50%;
          transform: translateY(-50%);
          color: #6b7280;
        }

        .data-table {
          width: 100%;
          border-collapse: collapse;
        }

        .data-table th {
          background: #f3f4f6;
          padding: 12px;
          text-align: left;
          font-weight: 600;
        }

        .data-table td {
          padding: 12px;
          border-bottom: 1px solid #e5e7eb;
        }

        .status-badge {
          padding: 4px 12px;
          border-radius: 20px;
          font-size: 12px;
          font-weight: 600;
        }

        .status-badge.active { background: #d1fae5; color: #065f46; }
        .status-badge.passive { background: #fed7aa; color: #92400e; }
        .status-badge.invested { background: #ddd6fe; color: #5b21b6; }
        .status-badge.left_investment { background: #fef3c7; color: #92400e; }
        .status-badge.pending_check { background: #cffafe; color: #155e75; }

        .tags {
          display: flex;
          gap: 5px;
          flex-wrap: wrap;
        }

        .tag {
          background: #e5e7eb;
          padding: 2px 8px;
          border-radius: 4px;
          font-size: 11px;
        }

        .alert-badge {
          background: #ef4444;
          color: white;
          padding: 2px 6px;
          border-radius: 4px;
          font-size: 11px;
          margin-left: 5px;
        }

        .action-buttons {
          display: flex;
          gap: 10px;
        }

        .btn {
          padding: 10px 20px;
          border: none;
          border-radius: 8px;
          cursor: pointer;
          font-weight: 500;
          transition: all 0.3s;
        }

        .btn-primary {
          background: #4f46e5;
          color: white;
        }

        .btn-primary:hover {
          background: #4338ca;
        }

        .btn-secondary {
          background: #6b7280;
          color: white;
        }

        .btn-warning {
          background: #f59e0b;
          color: white;
        }

        .btn-small {
          padding: 6px 12px;
          font-size: 12px;
        }

        .btn-icon {
          background: transparent;
          border: none;
          cursor: pointer;
          padding: 8px;
          border-radius: 8px;
          transition: all 0.3s;
        }

        .btn-icon:hover {
          background: #f3f4f6;
        }

        .modal {
          position: fixed;
          top: 0;
          left: 0;
          right: 0;
          bottom: 0;
          background: rgba(0,0,0,0.5);
          display: flex;
          align-items: center;
          justify-content: center;
          z-index: 1000;
        }

        .modal-content {
          background: white;
          border-radius: 12px;
          padding: 30px;
          width: 90%;
          max-width: 500px;
        }

        .modal-content h3 {
          margin-bottom: 20px;
        }

        .modal-content form {
          display: flex;
          flex-direction: column;
          gap: 15px;
        }

        .modal-content input,
        .modal-content select,
        .modal-content textarea {
          padding: 10px;
          border: 2px solid #e5e7eb;
          border-radius: 8px;
          font-size: 14px;
        }

        .modal-content textarea {
          min-height: 100px;
          resize: vertical;
        }

        .modal-actions {
          display: flex;
          gap: 10px;
          justify-content: flex-end;
          margin-top: 20px;
        }
      `}</style>
    </div>
  );
}

// ===== 13. KURULUM TALİMATLARI =====
/*
KURULUM ADIMLARI:

1. PROJE OLUŞTURMA:
   mkdir call-panel
   cd call-panel
   npm init -y

2. BAĞIMLILIKLARI YÜKLEME:
   npm install next react react-dom @prisma/client prisma bcryptjs jsonwebtoken cookie axios react-hot-toast react-icons date-fns recharts xlsx node-cron

3. DOSYA YAPISINI OLUŞTURMA:
   - Yukarıdaki tüm dosyaları ilgili klasörlere yerleştirin
   - .env.local dosyasını oluşturun ve veritabanı bilgilerini girin

4. POSTGRESQL KURULUMU:
   - cPanel'de PostgreSQL veritabanı oluşturun
   - Kullanıcı adı ve şifre belirleyin
   - .env.local dosyasında DATABASE_URL'i güncelleyin

5. VERITABANI ŞEMASINI OLUŞTURMA:
   npx prisma db push

6. VARSAYILAN VERİLERİ YÜKLEME:
   node prisma/seed.js

7. UYGULAMAYI ÇALIŞTIRMA:
   npm run build
   npm start

8. cPANEL'DE NODE.JS UYGULAMASI OLARAK YAYINLAMA:
   - Setup Node.js App'ten yeni uygulama oluşturun
   - Application root: call-panel
   - Application URL: yourdomain.com
   - Application startup file: node_modules/next/dist/bin/next
   - Run NPM Install butonuna tıklayın
   - Start App butonuna tıklayın

VARSAYILAN GİRİŞ BİLGİLERİ:
- Admin: admin / admin123
- Supervisor: supervisor / super123
- Operator: operator1 / op123

ÖZELLİKLER:
✓ 29 farklı operatör desteği
✓ Rol tabanlı yetkilendirme (Admin, Supervisor, Operator)
✓ Özel tag sistemi
✓ Yatırım takibi
✓ 3 gün sonra otomatik kontrol hatırlatması
✓ Yatırım bıraktı durumu
✓ Detaylı raporlama
✓ Bildirim sistemi
✓ Aktivite logları
✓ Excel export
*/
