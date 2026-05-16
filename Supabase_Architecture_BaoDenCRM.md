# KẾ HOẠCH TRIỂN KHAI & THIẾT KẾ KIẾN TRÚC SUPABASE - BÁO ĐEN LOGISTICS CRM

Tài liệu này trình bày toàn bộ giải pháp kỹ thuật trên nền tảng Supabase theo yêu cầu PRD và Design Brief của dự án CRM Báo Đen Logistics.

---

## 1. Cơ Chế Xác Thực & Quản Lý Tài Khoản (Authentication Flow)

Hệ thống hỗ trợ đồng thời đăng nhập bằng Email/Password và Google OAuth. 

**Quy trình Xác Thực (Auth Flow):**
1. **Đăng ký Email/Mật khẩu:** Người dùng nhập Email và Mật khẩu. Supabase Auth sẽ gửi email xác nhận. Sau khi xác nhận, tài khoản chính thức kích hoạt.
2. **Đăng nhập Google OAuth:** Người dùng click "Sign in with Google". Supabase chuyển hướng sang Google. Khi xác thực thành công, trả về phiên đăng nhập.
3. **Quên / Đặt lại mật khẩu:** Người dùng nhập email, Supabase gửi link magic/reset password có thời hạn. Người dùng click vào link để đổi mật khẩu.

**Xử lý Trùng lặp & Hợp nhất Tài khoản (Identity Linking):**
- **Logic:** Supabase có tính năng tự động liên kết tài khoản dựa trên Email. Khi bật tính năng `Link identities automatically` trong cài đặt Supabase Auth, hệ thống sẽ ưu tiên Email làm khóa chính.
- **Trường hợp 1: Có tài khoản Email/Password, đăng nhập bằng Google OAuth.** Hệ thống sẽ thấy trùng Email và tự động hợp nhất phiên bản Google vào tài khoản hiện tại. User vẫn giữ nguyên ID, quyền hạn và dữ liệu liên kết.
- **Trường hợp 2: Có tài khoản Google OAuth, sau đó cố gắng đăng ký lại bằng Email/Password.** Tùy cấu hình, hệ thống sẽ báo lỗi "Người dùng đã tồn tại" (để bảo mật), buộc người dùng dùng tính năng "Quên mật khẩu" để thiết lập mật khẩu cho tài khoản đó, thay vì tạo mới.
- **Trigger `handle_new_user`:** Chỉ chạy 1 lần khi bản ghi trong bảng `auth.users` được tạo. Dù đăng nhập bằng nhiều phương thức thì bản ghi `auth.users` chỉ có một (nếu chung email), đảm bảo chỉ có một bản ghi `profiles` duy nhất, và `user_id` không đổi.

---

## 2. Cơ Chế Phân Quyền (Authorization)

Hệ thống áp dụng 3 tầng truy cập (Role-Based Access Control) thông qua `Row Level Security (RLS)` của PostgreSQL:

1. **Admin (Quản trị viên):** Quyền xem, thêm, sửa, xóa toàn bộ hệ thống.
2. **Team Leader (Trưởng nhóm):** Xem được dữ liệu do bản thân và các nhân viên thuộc nhóm mình (`teams`, `team_members`) phụ trách.
3. **Sales / CSKH (Nhân viên):** Chỉ nhìn thấy dữ liệu của chính mình (nơi `sales_id = auth.uid()`). Không thể xem dữ liệu của đồng nghiệp hoặc nhóm khác.

**Lưu trữ Role:** Role (vai trò) được lưu trong trường `role` của bảng `profiles`. Mặc định khi một người đăng ký mới sẽ có role là `sales`. Admin có thể thay đổi thuộc tính này.
Để hệ thống không bị lỗi *infinite recursion* khi query bảng `profiles` trong chính RLS policy, một function với cờ `SECURITY DEFINER` được sử dụng (`get_my_role()`).

---

## 3. Cấu Trúc Bảng, Quan Hệ & Index (SQL Schema)

Dưới đây là đoạn mã SQL tạo toàn bộ các bảng, các liên kết khóa ngoại (Foreign Keys), và Indexes.

```sql
-- Kích hoạt Extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- 1. ĐỊNH NGHĨA ENUMS
CREATE TYPE user_role AS ENUM ('admin', 'team_leader', 'sales');
CREATE TYPE lead_segment AS ENUM ('Kim cương', 'Vàng', 'Bạc');
CREATE TYPE task_priority AS ENUM ('Cao', 'Trung bình', 'Thấp');
CREATE TYPE task_status AS ENUM ('Đang chờ xử lý', 'Đã hoàn thành', 'Quá hạn');

-- 2. TẠO CÁC BẢNG

-- Bảng Profiles
CREATE TABLE public.profiles (
    id UUID REFERENCES auth.users(id) ON DELETE CASCADE PRIMARY KEY,
    email TEXT UNIQUE NOT NULL,
    full_name TEXT,
    phone TEXT,
    avatar_url TEXT,
    role user_role DEFAULT 'sales'::user_role NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW() NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT NOW() NOT NULL
);

-- Bảng Teams (Nhóm kinh doanh)
CREATE TABLE public.teams (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    name TEXT NOT NULL,
    leader_id UUID REFERENCES public.profiles(id) ON DELETE SET NULL,
    created_at TIMESTAMPTZ DEFAULT NOW() NOT NULL
);

-- Bảng Team Members (Liên kết Nhóm - Người dùng)
CREATE TABLE public.team_members (
    team_id UUID REFERENCES public.teams(id) ON DELETE CASCADE,
    user_id UUID REFERENCES public.profiles(id) ON DELETE CASCADE,
    joined_at TIMESTAMPTZ DEFAULT NOW() NOT NULL,
    PRIMARY KEY (team_id, user_id)
);

-- Bảng Pipeline Stages (Giai đoạn Kanban)
CREATE TABLE public.pipeline_stages (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    name TEXT NOT NULL,
    position INTEGER NOT NULL,
    is_closed BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT NOW() NOT NULL
);

-- Bảng Leads (Khách hàng)
CREATE TABLE public.leads (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    full_name TEXT NOT NULL,
    company TEXT,
    phone TEXT,
    email TEXT,
    segment lead_segment DEFAULT 'Bạc'::lead_segment,
    expected_value BIGINT DEFAULT 0,
    pipeline_stage_id UUID REFERENCES public.pipeline_stages(id) ON DELETE RESTRICT,
    sales_id UUID REFERENCES public.profiles(id) ON DELETE SET NULL,
    created_at TIMESTAMPTZ DEFAULT NOW() NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT NOW() NOT NULL
);

-- Bảng Lead Tags (Dịch vụ quan tâm / Nhãn)
CREATE TABLE public.lead_tags (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    name TEXT NOT NULL UNIQUE,
    tag_type TEXT DEFAULT 'service', 
    created_at TIMESTAMPTZ DEFAULT NOW() NOT NULL
);

-- Bảng liên kết Leads - Tags
CREATE TABLE public.lead_tag_mappings (
    lead_id UUID REFERENCES public.leads(id) ON DELETE CASCADE,
    tag_id UUID REFERENCES public.lead_tags(id) ON DELETE CASCADE,
    PRIMARY KEY (lead_id, tag_id)
);

-- Bảng Lead Lists (Nhóm danh sách)
CREATE TABLE public.lead_lists (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    name TEXT NOT NULL,
    description TEXT,
    created_by UUID REFERENCES public.profiles(id) ON DELETE SET NULL,
    created_at TIMESTAMPTZ DEFAULT NOW() NOT NULL
);

-- Bảng liên kết Leads - Lists
CREATE TABLE public.lead_list_members (
    list_id UUID REFERENCES public.lead_lists(id) ON DELETE CASCADE,
    lead_id UUID REFERENCES public.leads(id) ON DELETE CASCADE,
    added_at TIMESTAMPTZ DEFAULT NOW() NOT NULL,
    PRIMARY KEY (list_id, lead_id)
);

-- Bảng Tasks (Công việc)
CREATE TABLE public.tasks (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    lead_id UUID REFERENCES public.leads(id) ON DELETE CASCADE,
    title TEXT NOT NULL,
    description TEXT,
    priority task_priority DEFAULT 'Trung bình'::task_priority,
    status task_status DEFAULT 'Đang chờ xử lý'::task_status,
    due_date TIMESTAMPTZ,
    assigned_to UUID REFERENCES public.profiles(id) ON DELETE SET NULL,
    created_at TIMESTAMPTZ DEFAULT NOW() NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT NOW() NOT NULL
);

-- Bảng Subscriptions / Payments (Thanh toán / Đặt cọc)
CREATE TABLE public.subscriptions (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    lead_id UUID REFERENCES public.leads(id) ON DELETE CASCADE,
    service_name TEXT NOT NULL,
    amount BIGINT NOT NULL,
    status TEXT DEFAULT 'Pending',
    payment_proof_url TEXT,
    created_by UUID REFERENCES public.profiles(id) ON DELETE SET NULL,
    created_at TIMESTAMPTZ DEFAULT NOW() NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT NOW() NOT NULL
);

-- Bảng Activity Logs (Nhật ký thao tác)
CREATE TABLE public.activity_logs (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    user_id UUID REFERENCES public.profiles(id) ON DELETE SET NULL,
    action TEXT NOT NULL, 
    entity_type TEXT NOT NULL, 
    entity_id UUID NOT NULL,
    details JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW() NOT NULL
);

-- 3. INDEXES
CREATE INDEX idx_leads_sales_id ON public.leads(sales_id);
CREATE INDEX idx_leads_pipeline_stage_id ON public.leads(pipeline_stage_id);
CREATE INDEX idx_leads_segment ON public.leads(segment);
CREATE INDEX idx_team_members_user_id ON public.team_members(user_id);
CREATE INDEX idx_team_members_team_id ON public.team_members(team_id);
CREATE INDEX idx_tasks_assigned_to ON public.tasks(assigned_to);
CREATE INDEX idx_tasks_lead_id ON public.tasks(lead_id);
CREATE INDEX idx_activity_logs_entity_id ON public.activity_logs(entity_id);
```

---

## 4. Triggers Tự Động & Helper Functions

```sql
-- Function Tự động cập nhật `updated_at`
CREATE OR REPLACE FUNCTION update_modified_column() RETURNS TRIGGER AS $$
BEGIN NEW.updated_at = NOW(); RETURN NEW; END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_profiles_upd BEFORE UPDATE ON public.profiles FOR EACH ROW EXECUTE FUNCTION update_modified_column();
CREATE TRIGGER trg_leads_upd BEFORE UPDATE ON public.leads FOR EACH ROW EXECUTE FUNCTION update_modified_column();
CREATE TRIGGER trg_tasks_upd BEFORE UPDATE ON public.tasks FOR EACH ROW EXECUTE FUNCTION update_modified_column();

-- Function tự tạo profile ngay khi đăng ký
CREATE OR REPLACE FUNCTION public.handle_new_user() RETURNS trigger AS $$
BEGIN
  INSERT INTO public.profiles (id, email, full_name, avatar_url, role)
  VALUES (
    NEW.id, NEW.email,
    COALESCE(NEW.raw_user_meta_data->>'full_name', NEW.raw_user_meta_data->>'name', ''),
    COALESCE(NEW.raw_user_meta_data->>'avatar_url', ''),
    'sales'::user_role
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();

-- Helper function cho RLS (Security Definer giúp bỏ qua vòng lặp đệ quy)
CREATE OR REPLACE FUNCTION public.get_my_role() RETURNS user_role AS $$
  SELECT role FROM public.profiles WHERE id = auth.uid() LIMIT 1;
$$ LANGUAGE sql SECURITY DEFINER;

CREATE OR REPLACE FUNCTION public.is_in_my_team(target_user_id UUID) RETURNS BOOLEAN AS $$
  SELECT EXISTS (
    SELECT 1 FROM public.team_members tm
    JOIN public.teams t ON t.id = tm.team_id
    WHERE t.leader_id = auth.uid() AND tm.user_id = target_user_id
  );
$$ LANGUAGE sql SECURITY DEFINER;
```

---

## 5. Row Level Security (RLS) Policies

```sql
-- Bật RLS cho các bảng nghiệp vụ
ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.leads ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.tasks ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.teams ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.team_members ENABLE ROW LEVEL SECURITY;

-- POLICIES BẢNG PROFILES
CREATE POLICY "View Profiles" ON public.profiles FOR SELECT USING (
    id = auth.uid() OR public.get_my_role() = 'admin' OR public.is_in_my_team(id)
);
CREATE POLICY "Update Own Profile" ON public.profiles FOR UPDATE USING (id = auth.uid());

-- POLICIES BẢNG LEADS (Lõi bảo mật)
CREATE POLICY "Select Leads" ON public.leads FOR SELECT USING (
    public.get_my_role() = 'admin' OR 
    sales_id = auth.uid() OR 
    public.is_in_my_team(sales_id)
);
CREATE POLICY "Insert Leads" ON public.leads FOR INSERT WITH CHECK (
    public.get_my_role() = 'admin' OR sales_id = auth.uid()
);
CREATE POLICY "Update Leads" ON public.leads FOR UPDATE USING (
    public.get_my_role() = 'admin' OR sales_id = auth.uid() OR public.is_in_my_team(sales_id)
);
CREATE POLICY "Delete Leads" ON public.leads FOR DELETE USING (public.get_my_role() = 'admin');

-- POLICIES BẢNG TASKS
CREATE POLICY "Select Tasks" ON public.tasks FOR SELECT USING (
    public.get_my_role() = 'admin' OR assigned_to = auth.uid() OR public.is_in_my_team(assigned_to)
);
CREATE POLICY "Insert Tasks" ON public.tasks FOR INSERT WITH CHECK (
    public.get_my_role() = 'admin' OR assigned_to = auth.uid() OR public.is_in_my_team(assigned_to)
);
CREATE POLICY "Update Tasks" ON public.tasks FOR UPDATE USING (
    public.get_my_role() = 'admin' OR assigned_to = auth.uid() OR public.is_in_my_team(assigned_to)
);
```

---

## 6. Dữ Liệu Mẫu Seed Data (Báo Đen Logistics)

```sql
-- 1. Chèn Pipeline Stages
INSERT INTO public.pipeline_stages (name, position, is_closed) VALUES
('Mới tiếp cận', 1, FALSE), ('Đang tư vấn', 2, FALSE), ('Gửi báo giá', 3, FALSE), 
('Thương lượng', 4, FALSE), ('Đã chốt đơn', 5, TRUE), ('Thất bại', 6, TRUE);

-- 2. Chèn Lead Tags (Dịch vụ)
INSERT INTO public.lead_tags (name, tag_type) VALUES
('Dịch vụ vận chuyển đường biển', 'service'),
('Dịch vụ khai báo hải quan', 'service'),
('Dịch vụ order chính ngạch', 'service'),
('Dịch vụ vận chuyển đường bộ', 'service'),
('Dịch vụ order tiểu ngạch', 'service'),
('Vận chuyển hàng không nhanh', 'service');

-- 3. Chèn Danh sách Phân Loại
INSERT INTO public.lead_lists (name, description) VALUES
('Khách sỉ nhập xưởng', 'Khách hàng nhập sỉ trực tiếp từ các xưởng sản xuất Quảng Châu, Thâm Quyến'),
('Khách Order Taobao lẻ', 'Các cá nhân kinh doanh online hoặc mua hộ lẻ qua Taobao, 1688, Tmall'),
('Doanh nghiệp nhập khẩu chính ngạch', 'Các công ty yêu cầu khai báo hải quan, xuất hóa đơn đỏ và giấy tờ kiểm định đầy đủ');

-- ==========================================
-- GHI CHÚ: Với Khách Hàng (Leads) & Công Việc (Tasks),
-- Trong thực tế chúng ta cần sử dụng UUID của Sales & Stage thực, tuy nhiên cấu trúc sẽ như sau:
-- ==========================================
/*
INSERT INTO public.leads (full_name, company, segment, expected_value) VALUES
('Nguyễn Văn An', 'Xưởng may mặc Hà Nội', 'Kim cương', 1500000000),
('Trần Thị Bích', 'Shop Mẹ & Bé Sài Gòn', 'Vàng', 350000000),
('Lê Minh Tiến', 'Công ty XNK Nam Việt', 'Kim cương', 800000000),
('Phạm Quang Hưng', 'Cá nhân (Order Taobao)', 'Bạc', 15000000),
('Hoàng Thu Trang', 'Cửa hàng Phụ kiện Giá sỉ', 'Vàng', 85000000),
('Đinh Công Mạnh', 'Đại lý Nội thất Thông minh', 'Kim cương', 1200000000);

INSERT INTO public.tasks (title, description, priority, due_date) VALUES
('Báo giá vận chuyển đường biển', 'Báo giá lô hàng 5 container thiết bị vệ sinh từ Thâm Quyến cho XNK Nam Việt', 'Cao', '2026-05-15 00:00:00+00'),
('Tư vấn khai báo hải quan', 'Hỗ trợ khách hàng Lê Minh Tiến về thủ tục C/O form E', 'Cao', '2026-05-16 00:00:00+00'),
('Gửi hợp đồng nguyên tắc', 'Gửi hợp đồng vận chuyển đường bộ cho Shop Mẹ & Bé', 'Trung bình', '2026-05-17 00:00:00+00');
*/
```
