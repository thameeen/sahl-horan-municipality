┌────────────────────────────────────────────────────────────────────────┐
│                        واجهات المستخدم (Client Tier)                  │
├───────────────────────────────────┬────────────────────────────────────┤
│  بوابة وتطبيق المواطن (PWA/RTL)    │   لوحة التحكم البلدية (Admin CRM)  │
│  - رفع البلاغات وتحديد الموقع     │   - فرز وتعيين الفرق الميدانية     │
│  - التتبع بالرقم المرجعي          │   - توثيق الإنجاز (قبل / بعد)      │
│  - خريطة البلاغات العامة المحمية   │   - إدارة النطاقات الجغرافية       │
│  - مركز الإشعارات والخدمات        │   - إحصائيات الأداء وتصدير التقارير│
└─────────────────┬─────────────────┴──────────────────┬─────────────────┘
                  │ HTTPS / WSS                        │ HTTPS / WSS
┌─────────────────▼────────────────────────────────────▼─────────────────┐
│                      طبقة التطبيق والواجهات البرمجية                   │
│      Next.js (App Router) / Node.js API + WebSocket Service            │
├────────────────────────────────────────────────────────────────────────┤
│ • محرك التحقق الجغرافي (Sahl Horan Geofence Validator)                 │
│ • محرك إخفاء البيانات الحساسة (PII Sanitizer & Data Redaction)         │
│ • نظام الصلاحيات المبني على الأدوار (RBAC: Citizen, Staff, Supervisor) │
│ • وحدة إرسال الإشعارات متعددة القنوات (Web Push / SSE / SMS)          │
└─────────────────┬────────────────────────────────────┬─────────────────┘
                  │                                    │
┌─────────────────▼─────────────────┐ ┌────────────────▼─────────────────┐
│   قاعدة البيانات والتحليل المكاني │ │       تخزين الملفات والوسائط    │
│    PostgreSQL + PostGIS Extension │ │     S3-Compatible Object Store   │
│   - مضلعات المناطق الأربعة        │ │     - ضغط الصور وتطهير الـ EXIF │
│   - سياسات الأمان Row Level (RLS) │ │     - صور ما قبل وما بعد الإصلاح │
└───────────────────────────────────┘ └──────────────────────────────────┘
-- تفعيل ملحق العمليات الجغرافية
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- 1. جدول المناطق الجغرافية المعتمدة لسهل حوران
CREATE TABLE municipal_districts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name_ar VARCHAR(50) NOT NULL UNIQUE, -- الطرة، الشجرة، الذنيبة، عمراوة
    name_en VARCHAR(50) NOT NULL,
    boundary GEOMETRY(Polygon, 4326) NOT NULL, -- مضلع الحدود الجغرافية
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 2. جدول المستخدمين والأدوار
CREATE TYPE user_role AS ENUM ('citizen', 'field_staff', 'supervisor', 'admin');

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    national_id VARCHAR(15) UNIQUE,
    full_name VARCHAR(120) NOT NULL,
    phone_number VARCHAR(20) UNIQUE NOT NULL,
    role user_role DEFAULT 'citizen',
    district_id UUID REFERENCES municipal_districts(id),
    department VARCHAR(50), -- للأشغال، النظافة، الإنارة في حال كان موظفاً
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 3. جدول تصنيفات البلاغات البلدية
CREATE TABLE report_categories (
    id SERIAL PRIMARY KEY,
    key VARCHAR(50) UNIQUE NOT NULL,
    title_ar VARCHAR(100) NOT NULL,
    icon_name VARCHAR(50) NOT NULL,
    sla_hours INT DEFAULT 48, -- زمن الاستجابة المستهدف (بالساعات)
    department VARCHAR(50) NOT NULL
);

-- 4. جدول البلاغات الأساسي
CREATE TYPE report_status AS ENUM (
    'received',      -- تم الاستلام
    'under_review',  -- قيد المراجعة
    'in_progress',   -- قيد المعالجة
    'resolved',      -- تمت المعالجة
    'transferred'    -- تم تحويله لجهة أخرى
);

CREATE TABLE municipal_reports (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tracking_number VARCHAR(16) UNIQUE NOT NULL, -- مثال: SH-2026-X89B
    citizen_id UUID REFERENCES users(id) ON DELETE SET NULL,
    category_id INT REFERENCES report_categories(id),
    district_id UUID REFERENCES municipal_districts(id) NOT NULL,
    location GEOMETRY(Point, 4326) NOT NULL,
    address_description TEXT,
    description TEXT NOT NULL,
    status report_status DEFAULT 'received',
    is_public BOOLEAN DEFAULT TRUE, -- هل يُعرض على الخريطة العامة؟
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- فهرس مكاني لتسريع استعلامات الخرائط
CREATE INDEX idx_reports_location ON municipal_reports USING GIST(location);

-- 5. جدول وسائط البلاغ (قبل وبعد الصيانة)
CREATE TABLE report_media (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    report_id UUID REFERENCES municipal_reports(id) ON DELETE CASCADE,
    media_url TEXT NOT NULL,
    media_type VARCHAR(20) DEFAULT 'image',
    phase VARCHAR(10) CHECK (phase IN ('before', 'after')),
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 6. سجل تتبع الحالات (Audit Log) لتوثيق كل حركة
CREATE TABLE report_status_history (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    report_id UUID REFERENCES municipal_reports(id) ON DELETE CASCADE,
    old_status report_status,
    new_status report_status NOT NULL,
    notes TEXT,
    actor_id UUID REFERENCES users(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 7. جدول الإشعارات الموجهة
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(150) NOT NULL,
    body TEXT NOT NULL,
    link_url TEXT,
    is_read BOOLEAN DEFAULT FALSE,
    notification_type VARCHAR(50) DEFAULT 'system',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
// lib/geofencing.ts

export interface Coordinate {
  latitude: number;
  longitude: number;
}

// المضلع الجغرافي الشامل لمناطق سهل حوران (الطرة، الشجرة، الذنيبة، عمراوة)
export const SAHL_HOURAN_BBOX = {
  minLat: 32.5800,
  maxLat: 32.7200,
  minLng: 35.8500,
  maxLng: 36.0500,
  center: [32.6500, 35.9500] as [number, number]
};

export const MUNICIPAL_DISTRICTS = [
  { id: 'turrah', nameAr: 'الطرة', center: [32.6120, 35.9910] },
  { id: 'shajarah', nameAr: 'الشجرة', center: [32.6515, 35.9420] },
  { id: 'dhanibah', nameAr: 'الذنيبة', center: [32.6780, 35.9210] },
  { id: 'amrawah', nameAr: 'عمراوة', center: [32.6840, 35.8850] }
];

export function isWithinSahlHouran(coord: Coordinate): boolean {
  return (
    coord.latitude >= SAHL_HOURAN_BBOX.minLat &&
    coord.latitude <= SAHL_HOURAN_BBOX.maxLat &&
    coord.longitude >= SAHL_HOURAN_BBOX.minLng &&
    coord.longitude <= SAHL_HOURAN_BBOX.maxLng
  );
}
// app/api/reports/route.ts
import { NextResponse } from 'next/server';
import { isWithinSahlHouran } from '@/lib/geofencing';
import { db } from '@/lib/db'; // إتصال قاعدة البيانات

export async function POST(req: Request) {
  try {
    const body = await req.json();
    const { categoryId, districtId, latitude, longitude, description, images, citizenId } = body;

    // 1. التحقق من الحدود الجغرافية
    if (!isWithinSahlHouran({ latitude, longitude })) {
      return NextResponse.json(
        { error: 'الموقع المحدد يقع خارج النطاق الجغرافي لبلدية سهل حوران (الطرة، الشجرة، الذنيبة، عمراوة).' },
        { status: 400 }
      );
    }

    // 2. توليد رقم تتبع فريد بنمط رسمي (مثال: SH-26-8942)
    const randomSuffix = Math.floor(1000 + Math.random() * 9000);
    const trackingNumber = `SH-${new Date().getFullYear().toString().slice(-2)}-${randomSuffix}`;

    // 3. تخزين البلاغ داخل معاملة بنكية واحدة (Transaction)
    const newReport = await db.$transaction(async (tx) => {
      const report = await tx.municipalReport.create({
        data: {
          trackingNumber,
          categoryId,
          districtId,
          latitude,
          longitude,
          description,
          citizenId,
          status: 'received'
        }
      });

      // إدراج الصور المرتبطة (مرحلة ما قبل الإصلاح)
      if (images && images.length > 0) {
        await tx.reportMedia.createMany({
          data: images.map((url: string) => ({
            reportId: report.id,
            mediaUrl: url,
            phase: 'before'
          }))
        });
      }

      // توثيق الخطوة الأولى في السجل الزمني
      await tx.reportStatusHistory.create({
        data: {
          reportId: report.id,
          newStatus: 'received',
          notes: 'تم استلام البلاغ آلياً عبر تطبيق بلدية سهل حوران الذكية'
        }
      });

      return report;
    });

    return NextResponse.json({
      success: true,
      trackingNumber: newReport.trackingNumber,
      message: 'تم إرسال البلاغ بنجاح وتوليد رقم التتبع'
    }, { status: 201 });

  } catch (error) {
    return NextResponse.json({ error: 'تعذر معالجة البلاغ حالياً' }, { status: 500 });
  }
}

// عرض البلاغات العامة على الخريطة مع حماية البيانات الشخصية PII
export async function GET(req: Request) {
  const { searchParams } = new URL(req.url);
  const districtId = searchParams.get('districtId');

  const reports = await db.municipalReport.findMany({
    where: {
      isPublic: true,
      ...(districtId ? { districtId } : {})
    },
    select: {
      id: true,
      trackingNumber: true,
      latitude: true,
      longitude: true,
      status: true,
      createdAt: true,
      category: { select: { titleAr: true, iconName: true } },
      district: { select: { nameAr: true } },
      media: { where: { phase: 'before' }, take: 1, select: { mediaUrl: true } }
      // عدم تضمين أي بيانات تخص اسم المواطن أو هاتفه نهائياً
    }
  });

  return NextResponse.json(reports);
}
// components/CitizenReportForm.tsx
'use client';

import React, { useState } from 'react';
import { Camera, MapPin, CheckCircle, AlertTriangle, ArrowRight } from 'lucide-react';
import { SAHL_HOURAN_BBOX, MUNICIPAL_DISTRICTS } from '@/lib/geofencing';

const CATEGORIES = [
  { id: 1, name: 'تراكم النفايات والحاويات', icon: '🗑️' },
  { id: 2, name: 'الحفر وهبوطات الطرق', icon: '🚧' },
  { id: 3, name: 'أعطال إنارة الشوارع', icon: '💡' },
  { id: 4, name: 'تصريف مياه الأمطار والمناهل', icon: '🌧️' },
  { id: 5, name: 'الأرصفة والحدائق والمرافق', icon: '🌳' },
  { id: 6, name: 'تعديات على الساحات والشوارع', icon: '⚠️' }
];

export default function CitizenReportForm() {
  const [step, setStep] = useState(1);
  const [imagePreview, setImagePreview] = useState<string | null>(null);
  const [selectedCategory, setSelectedCategory] = useState<number | null>(null);
  const [selectedDistrict, setSelectedDistrict] = useState<string>('turrah');
  const [coords, setCoords] = useState<{ lat: number; lng: number } | null>(null);
  const [description, setDescription] = useState('');
  const [loading, setLoading] = useState(false);
  const [submittedCode, setSubmittedCode] = useState<string | null>(null);
  const [errorMsg, setErrorMsg] = useState<string | null>(null);

  // التقاط الموقع مع حصر التحقق داخل سهل حوران
  const handleGetLocation = () => {
    if (!navigator.geolocation) {
      setErrorMsg('خدمة تحديد المواقع غير مدعومة على متصفحك.');
      return;
    }

    navigator.geolocation.getCurrentPosition(
      (pos) => {
        const { latitude, longitude } = pos.coords;
        if (
          latitude < SAHL_HOURAN_BBOX.minLat || latitude > SAHL_HOURAN_BBOX.maxLat ||
          longitude < SAHL_HOURAN_BBOX.minLng || longitude > SAHL_HOURAN_BBOX.maxLng
        ) {
          setErrorMsg('أنت حالياً خارج النطاق الجغرافي لبلدية سهل حوران. يمكنك تحديد الموقع يدوياً على الخريطة.');
          return;
        }
        setCoords({ lat: latitude, lng: longitude });
        setErrorMsg(null);
      },
      () => {
        setErrorMsg('تعذر جلب موقعك بدقة، يرجى اختياره من خيارات المناطق المتاحة.');
      },
      { enableHighAccuracy: true, timeout: 8000 }
    );
  };

  const handleImageCapture = (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (file) {
      const reader = new FileReader();
      reader.onloadend = () => setImagePreview(reader.result as string);
      reader.readAsDataURL(file);
    }
  };

  const handleSubmit = async () => {
    if (!selectedCategory || !coords) {
      setErrorMsg('يرجى التأكد من اختيار نوع البلاغ وموقع المشكلة.');
      return;
    }

    setLoading(true);
    setErrorMsg(null);

    try {
      const res = await fetch('/api/reports', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          categoryId: selectedCategory,
          districtId: selectedDistrict,
          latitude: coords.lat,
          longitude: coords.lng,
          description,
          images: imagePreview ? [imagePreview] : []
        })
      });

      const data = await res.json();
      if (!res.ok) throw new Error(data.error);

      setSubmittedCode(data.trackingNumber);
      setStep(4); // الانتقال لشاشة النجاح
    } catch (err: any) {
      setErrorMsg(err.message || 'حدث خطأ في الاتصال، يرجى المحاولة لاحقاً');
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="max-w-2xl mx-auto p-4 bg-white rounded-2xl shadow-sm border border-stone-200" dir="rtl">
      {/* رأس الصفحة */}
      <div className="border-b border-stone-100 pb-4 mb-6">
        <span className="text-xs font-semibold px-2.5 py-1 bg-emerald-50 text-emerald-800 rounded-full">
          نظام الاستجابة البلدية السريعة
        </span>
        <h1 className="text-2xl font-bold text-stone-900 mt-2">تسجيل بلاغ خدمي</h1>
        <p className="text-sm text-stone-500">بلدية سهل حوران: الطرة • الشجرة • الذنيبة • عمراوة</p>
      </div>

      {errorMsg && (
        <div className="mb-4 p-3.5 bg-rose-50 border border-rose-200 text-rose-800 rounded-xl text-sm flex items-center gap-2">
          <AlertTriangle className="w-5 h-5 flex-shrink-0" />
          <span>{errorMsg}</span>
        </div>
      )}

      {/* الخطوة 1: توثيق المشكلة بالصورة وتحديد التصنيف */}
      {step === 1 && (
        <div className="space-y-6">
          <div>
            <label className="block text-sm font-medium text-stone-700 mb-2">1. تصوير أو إرفاق صورة العطل</label>
            <div className="relative border-2 border-dashed border-stone-300 rounded-xl p-6 text-center hover:bg-stone-50 transition cursor-pointer">
              <input
                type="file"
                accept="image/*"
                capture="environment"
                onChange={handleImageCapture}
                className="absolute inset-0 opacity-0 cursor-pointer w-full h-full"
              />
              {imagePreview ? (
                <img src={imagePreview} alt="معاينة البلاغ" className="mx-auto max-h-56 rounded-lg object-cover" />
              ) : (
                <div className="flex flex-col items-center">
                  <Camera className="w-10 h-10 text-stone-400 mb-2" />
                  <span className="text-sm font-medium text-stone-600">اضغط لالتقاط صورة مباشرة أو رفعها من المعرض</span>
                  <span className="text-xs text-stone-400 mt-1">تساعد الصورة على سرعة وصول وتجهيز فرق الصيانة المختصة</span>
                </div>
              )}
            </div>
          </div>

          <div>
            <label className="block text-sm font-medium text-stone-700 mb-2">2. نوع المشكلة البلدية</label>
            <div className="grid grid-cols-2 gap-3">
              {CATEGORIES.map((cat) => (
                <button
                  key={cat.id}
                  type="button"
                  onClick={() => setSelectedCategory(cat.id)}
                  className={`p-3.5 rounded-xl border text-right transition flex items-center gap-3 ${
                    selectedCategory === cat.id
                      ? 'border-emerald-600 bg-emerald-50 text-emerald-950 font-medium'
                      : 'border-stone-200 hover:border-stone-300 bg-white'
                  }`}
                >
                  <span className="text-2xl">{cat.icon}</span>
                  <span className="text-sm">{cat.name}</span>
                </button>
              ))}
            </div>
          </div>

          <button
            disabled={!selectedCategory}
            onClick={() => setStep(2)}
            className="w-full py-3.5 bg-stone-900 text-white rounded-xl font-medium hover:bg-stone-800 disabled:opacity-50 transition"
          >
            المتابعة لتحديد الموقع
          </button>
        </div>
      )}

      {/* الخطوة 2: تحديد الموقع الجغرافي محلياً */}
      {step === 2 && (
        <div className="space-y-6">
          <div>
            <label className="block text-sm font-medium text-stone-700 mb-2">البلدة / المنطقة المستهدفة</label>
            <div className="grid grid-cols-4 gap-2">
              {MUNICIPAL_DISTRICTS.map((d) => (
                <button
                  key={d.id}
                  type="button"
                  onClick={() => {
                    setSelectedDistrict(d.id);
                    setCoords({ lat: d.center[0], lng: d.center[1] });
                  }}
                  className={`py-2 px-3 text-center rounded-lg border text-sm font-medium transition ${
                    selectedDistrict === d.id
                      ? 'bg-stone-900 text-white border-stone-900'
                      : 'border-stone-200 text-stone-700 hover:border-stone-300'
                  }`}
                >
                  {d.nameAr}
                </button>
              ))}
            </div>
          </div>

          <div className="p-4 bg-stone-50 border border-stone-200 rounded-xl space-y-3">
            <button
              type="button"
              onClick={handleGetLocation}
              className="w-full flex items-center justify-center gap-2 py-3 bg-white border border-stone-300 text-stone-800 rounded-lg text-sm font-medium hover:bg-stone-100 transition shadow-sm"
            >
              <MapPin className="w-4 h-4 text-emerald-600" />
              تحديد موقعي الدقيق حالياً عبر نظام التموضع (GPS)
            </button>
            {coords && (
              <p className="text-xs text-stone-500 text-center font-mono">
                الإحداثيات المثبتة: {coords.lat.toFixed(5)}, {coords.lng.toFixed(5)}
              </p>
            )}
          </div>

          <div>
            <label className="block text-sm font-medium text-stone-700 mb-2">وصف مختصر أو علامة مميزة</label>
            <textarea
              rows={3}
              value={description}
              onChange={(e) => setDescription(e.target.value)}
              placeholder="مثال: بالقرب من مدرسة الطرة الثانوية للبنين، الشارع الرئيسي..."
              className="w-full p-3 border border-stone-200 rounded-xl text-sm focus:outline-none focus:ring-2 focus:ring-stone-900"
            />
          </div>

          <div className="flex gap-3">
            <button
              type="button"
              onClick={() => setStep(1)}
              className="w-1/3 py-3 border border-stone-200 rounded-xl text-stone-700 font-medium hover:bg-stone-50"
            >
              السابق
            </button>
            <button
              type="button"
              disabled={loading || !coords}
              onClick={handleSubmit}
              className="w-2/3 py-3 bg-stone-900 text-white rounded-xl font-medium hover:bg-stone-800 disabled:opacity-50"
            >
              {loading ? 'جاري الإرسال...' : 'تأكيد وإرسال البلاغ'}
            </button>
          </div>
        </div>
      )}

      {/* الخطوة 3: بطاقة التأكيد والرقم المرجعي على النمط الياباني */}
      {step === 4 && (
        <div className="text-center py-8 space-y-4">
          <div className="w-16 h-16 bg-emerald-100 text-emerald-600 rounded-full flex items-center justify-center mx-auto">
            <CheckCircle className="w-10 h-10" />
          </div>
          <h2 className="text-xl font-bold text-stone-900">تم تسجيل بلاغك بنجاح</h2>
          <p className="text-sm text-stone-600 max-w-md mx-auto">
            تم توجيه البلاغ مباشرة إلى شعبة العمليات الميدانية في بلدية سهل حوران. يمكنك تتبع تطور المعالجة في أي وقت عبر الرقم التالي:
          </p>

          <div className="p-4 bg-stone-100 rounded-xl max-w-xs mx-auto border border-stone-200">
            <span className="text-xs text-stone-500 block mb-1">رقم التتبع المرجعي</span>
            <span className="text-2xl font-mono font-bold tracking-wider text-stone-900">{submittedCode}</span>
          </div>

          <div className="pt-4 flex justify-center gap-3">
            <button
              onClick={() => {
                setStep(1);
                setImagePreview(null);
                setSelectedCategory(null);
                setSubmittedCode(null);
              }}
              className="px-6 py-2.5 bg-stone-900 text-white rounded-xl text-sm font-medium hover:bg-stone-800"
            >
              تسجيل بلاغ جديد
            </button>
            <button
              onClick={() => window.location.href = `/tracking?code=${submittedCode}`}
              className="px-6 py-2.5 border border-stone-200 text-stone-700 rounded-xl text-sm font-medium hover:bg-stone-50"
            >
              الانتقال لصفحة المتابعة
            </button>
          </div>
        </div>
      )}
    </div>
  );
}
// app/admin/reports/[id]/page.tsx
'use client';

import React, { useState } from 'react';
import { CheckCircle2, Clock, Truck, ShieldAlert, ArrowLeftRight } from 'lucide-react';

interface ReportDetail {
  id: string;
  trackingNumber: string;
  categoryTitle: string;
  districtName: string;
  description: string;
  status: 'received' | 'under_review' | 'in_progress' | 'resolved' | 'transferred';
  beforeImage: string;
  createdAt: string;
}

export default function AdminReportDetails({ report }: { report: ReportDetail }) {
  const [currentStatus, setCurrentStatus] = useState(report.status);
  const [afterImage, setAfterImage] = useState<string | null>(null);
  const [actionNotes, setActionNotes] = useState('');
  const [isUpdating, setIsUpdating] = useState(false);

  const handleUpdateStatus = async (newStatus: string) => {
    setIsUpdating(true);
    try {
      const res = await fetch(`/api/admin/reports/${report.id}/status`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          status: newStatus,
          notes: actionNotes,
          afterImage: afterImage
        })
      });

      if (res.ok) {
        setCurrentStatus(newStatus as any);
        alert('تم تحديث حالة البلاغ بنجاح وإشعار المواطن');
      }
    } catch (e) {
      alert('حدث خطأ أثناء حفظ التحديث');
    } finally {
      setIsUpdating(false);
    }
  };

  return (
    <div className="max-w-4xl mx-auto p-6 bg-white rounded-2xl border border-stone-200" dir="rtl">
      {/* تفاصيل الترويسة */}
      <div className="flex justify-between items-center pb-6 border-b border-stone-200">
        <div>
          <span className="text-xs font-mono font-bold text-stone-500 bg-stone-100 px-3 py-1 rounded-md">
            {report.trackingNumber}
          </span>
          <h1 className="text-xl font-bold text-stone-900 mt-2">{report.categoryTitle} - منطقة {report.districtName}</h1>
          <p className="text-xs text-stone-500 mt-1">وقت التسجيل: {new Date(report.createdAt).toLocaleString('ar-JO')}</p>
        </div>
        <div className="text-left">
          <span className="inline-block px-3 py-1 text-xs font-bold rounded-full bg-amber-50 text-amber-800 border border-amber-200">
            الحالة: {currentStatus}
          </span>
        </div>
      </div>

      {/* المقارنة البصرية (قبل / بعد المعالجة) - النموذج الياباني */}
      <div className="grid grid-cols-1 md:grid-cols-2 gap-6 my-6">
        <div>
          <h3 className="text-sm font-bold text-stone-700 mb-2">توثيق الحالة عند الاستلام (قبل)</h3>
          <img src={report.beforeImage || '/placeholder-issue.jpg'} alt="صورة المشكلة" className="w-full h-56 object-cover rounded-xl border border-stone-200" />
          <p className="text-xs text-stone-500 mt-2 p-2 bg-stone-50 rounded-lg">{report.description || 'لا يوجد وصف إضافي'}</p>
        </div>

        <div>
          <h3 className="text-sm font-bold text-stone-700 mb-2">توثيق الإنجاز والصيانة (بعد)</h3>
          {afterImage ? (
            <img src={afterImage} alt="صورة بعد الإصلاح" className="w-full h-56 object-cover rounded-xl border border-emerald-300" />
          ) : (
            <div className="border-2 border-dashed border-stone-300 h-56 rounded-xl flex flex-col items-center justify-center p-4 text-center">
              <span className="text-xs text-stone-500 mb-2">يجب على الموظف إرفاق صورة الإنجاز قبل تحويل البلاغ إلى "تمت المعالجة"</span>
              <input
                type="file"
                accept="image/*"
                onChange={(e) => {
                  const file = e.target.files?.[0];
                  if (file) {
                    const reader = new FileReader();
                    reader.onloadend = () => setAfterImage(reader.result as string);
                    reader.readAsDataURL(file);
                  }
                }}
                className="text-xs"
              />
            </div>
          )}
        </div>
      </div>

      {/* إجراءات تغيير الحالة */}
      <div className="p-4 bg-stone-50 rounded-xl border border-stone-200 space-y-4">
        <h4 className="text-sm font-bold text-stone-800">تحديث إجراءات العمل البلدي</h4>
        <textarea
          rows={2}
          value={actionNotes}
          onChange={(e) => setActionNotes(e.target.value)}
          placeholder="إضافة ملاحظات الإجراء (مثل: تم تحريك قلاب وضاغطة إلى الشارع، أو صيانة وحدة الإنارة برقم عمود 41)..."
          className="w-full p-2.5 bg-white border border-stone-200 rounded-lg text-sm"
        />

        <div className="flex flex-wrap gap-2">
          <button
            onClick={() => handleUpdateStatus('under_review')}
            className="px-3.5 py-2 bg-white border border-stone-300 rounded-lg text-xs font-semibold hover:bg-stone-100"
          >
            تحويل إلى قيد المراجعة
          </button>
          <button
            onClick={() => handleUpdateStatus('in_progress')}
            className="px-3.5 py-2 bg-blue-50 text-blue-700 border border-blue-200 rounded-lg text-xs font-semibold hover:bg-blue-100"
          >
            إحالة للفريق الميداني (قيد المعالجة)
          </button>
          <button
            onClick={() => handleUpdateStatus('transferred')}
            className="px-3.5 py-2 bg-purple-50 text-purple-700 border border-purple-200 rounded-lg text-xs font-semibold hover:bg-purple-100"
          >
            تحويل لجهة خارجية (شركة مياه اليرموك / كهرباء إربد)
          </button>
          <button
            disabled={!afterImage}
            onClick={() => handleUpdateStatus('resolved')}
            className="px-4 py-2 bg-emerald-600 text-white rounded-lg text-xs font-semibold hover:bg-emerald-700 disabled:opacity-50"
          >
            إغلاق البلاغ: تمت المعالجة بنجاح
          </button>
        </div>
      </div>
    </div>
  );
}
// data/wasteSchedule.ts
export interface DistrictSchedule {
  districtKey: string;
  districtNameAr: string;
  days: string[];
  collectionPeriod: 'صباحي' | 'مسائي';
  supervisorContact: string;
  notes: string;
}

export const SAHL_HOURAN_WASTE_SCHEDULE: DistrictSchedule[] = [
  {
    districtKey: 'turrah',
    districtNameAr: 'الطرة',
    days: ['السبت', 'الإثنين', 'الأربعاء'],
    collectionPeriod: 'صباحي',
    supervisorContact: '02-738-XXXX',
    notes: 'القطاع الشمالي والشارع العام يبدأ المرور من الساعة 7:00 صباحاً'
  },
  {
    districtKey: 'shajarah',
    districtNameAr: 'الشجرة',
    days: ['الأحد', 'الثلاثاء', 'الخميس'],
    collectionPeriod: 'صباحي',
    supervisorContact: '02-738-XXXX',
    notes: 'يشمل الأحياء الداخلية ومحيط المدارس'
  },
  {
    districtKey: 'dhanibah',
    districtNameAr: 'الذنيبة',
    days: ['السبت', 'الثلاثاء'],
    collectionPeriod: 'صباحي',
    supervisorContact: '02-738-XXXX',
    notes: 'يشمل مسار وادي الشلالة والحي الغربي'
  },
  {
    districtKey: 'amrawah',
    districtNameAr: 'عمراوة',
    days: ['الأحد', 'الأربعاء'],
    collectionPeriod: 'صباحي',
    supervisorContact: '02-738-XXXX',
    notes: 'المنطقة السكنية ومثلث عمراوة'
  }
];
// components/AccessibilityPreferences.tsx
'use client';

import React, { createContext, useContext, useState, useEffect } from 'react';

type ContrastMode = 'default' | 'high' | 'soft';
type TextScale = 'normal' | 'large' | 'extra-large';

interface UserPreferences {
  contrast: ContrastMode;
  textScale: TextScale;
  setContrast: (c: ContrastMode) => void;
  setTextScale: (t: TextScale) => void;
}

const PreferencesContext = createContext<UserPreferences | null>(null);

export function PreferencesProvider({ children }: { children: React.ReactNode }) {
  const [contrast, setContrast] = useState<ContrastMode>('default');
  const [textScale, setTextScale] = useState<TextScale>('normal');

  useEffect(() => {
    const root = document.documentElement;
    // تطبيق الفئات الهادئة دون وسوم تمييزية
    root.setAttribute('data-contrast-mode', contrast);
    root.setAttribute('data-text-scale', textScale);
  }, [contrast, textScale]);

  return (
    <PreferencesContext.Provider value={{ contrast, textScale, setContrast, setTextScale }}>
      <div className={`app-wrapper text-scale-${textScale} contrast-${contrast}`}>
        {children}
      </div>
    </PreferencesContext.Provider>
  );
}

export function AccessibilityToolbar() {
  const ctx = useContext(PreferencesContext);
  if (!ctx) return null;

  return (
    <div className="flex items-center gap-4 p-2 bg-stone-100 rounded-lg text-xs text-stone-700" dir="rtl">
      <span className="font-medium">خيارات العرض والقراءة:</span>
      <div className="flex gap-1">
        <button
          onClick={() => ctx.setTextScale('normal')}
          className={`px-2 py-1 rounded ${ctx.textScale === 'normal' ? 'bg-stone-900 text-white' : 'bg-white'}`}
        >
          خط قياسي
        </button>
        <button
          onClick={() => ctx.setTextScale('large')}
          className={`px-2 py-1 rounded ${ctx.textScale === 'large' ? 'bg-stone-900 text-white' : 'bg-white'}`}
        >
          خط مكبّر
        </button>
        <button
          onClick={() => ctx.setTextScale('extra-large')}
          className={`px-2 py-1 rounded ${ctx.textScale === 'extra-large' ? 'bg-stone-900 text-white' : 'bg-white'}`}
        >
          خط عريض
        </button>
      </div>

      <div className="flex gap-1">
        <button
          onClick={() => ctx.setContrast('default')}
          className={`px-2 py-1 rounded ${ctx.contrast === 'default' ? 'bg-stone-900 text-white' : 'bg-white'}`}
        >
          ألوان قياسية
        </button>
        <button
          onClick={() => ctx.setContrast('high')}
          className={`px-2 py-1 rounded ${ctx.contrast === 'high' ? 'bg-stone-900 text-white' : 'bg-white'}`}
        >
          تباين عالي للوضوح
        </button>
      </div>
    </div>
  );
}
