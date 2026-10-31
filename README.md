# XẾP LỊCH TỰ ĐỘNG

Ứng dụng web/PWA quản lý lịch làm theo tuần, công việc nhiều màu, thêm/sửa/xóa ca, kéo thả, kiểm tra quy tắc, xuất lịch và đồng bộ Supabase.

## Chạy thử và triển khai

GitHub Pages: vào Settings → Pages → Deploy from a branch → chọn branch main và thư mục /(root). Chờ xuất bản rồi mở URL Pages. Trên iPhone dùng Safari → Chia sẻ → Thêm vào màn hình chính.

## Cấu hình Supabase

Project URL và Publishable key đã được đặt trong index.html. Không sử dụng Secret key hoặc service_role trong frontend.

Chạy toàn bộ SQL bên dưới trong Supabase → SQL Editor:

```sql
create table if not exists public.planner_data (
  user_id uuid primary key references auth.users(id) on delete cascade,
  data jsonb not null default '{"jobs":[],"shifts":[],"settings":{}}'::jsonb,
  updated_at timestamptz not null default now()
);
alter table public.planner_data enable row level security;
drop policy if exists "Users can read own planner" on public.planner_data;
create policy "Users can read own planner" on public.planner_data for select to authenticated using (auth.uid() = user_id);
drop policy if exists "Users can insert own planner" on public.planner_data;
create policy "Users can insert own planner" on public.planner_data for insert to authenticated with check (auth.uid() = user_id);
drop policy if exists "Users can update own planner" on public.planner_data;
create policy "Users can update own planner" on public.planner_data for update to authenticated using (auth.uid() = user_id) with check (auth.uid() = user_id);

create or replace function public.merge_planner_data(p_data jsonb)
returns void language plpgsql security invoker set search_path = public as $$
declare v_uid uuid := auth.uid();
begin
  if v_uid is null then raise exception 'Authentication required'; end if;
  insert into public.planner_data(user_id,data,updated_at)
  values (v_uid,coalesce(p_data,'{}'::jsonb),now())
  on conflict (user_id) do update set data=excluded.data,updated_at=now();
end;
$$;
revoke all on function public.merge_planner_data(jsonb) from public;
grant execute on function public.merge_planner_data(jsonb) to authenticated;
```

Sau đó vào Database → Publications, bật Realtime cho bảng planner_data.

## Chức năng

- Lịch tuần, chuyển tuần.
- Thêm/sửa/xóa ca, công việc, ghi chú.
- Kéo thả ca sang ngày khác hoặc vào vùng xóa.
- Quản lý tên và màu công việc.
- Cài đặt thời gian nghỉ, mục tiêu giờ/tuần, giới hạn ca/ngày.
- Kiểm tra điều kiện lịch.
- Xuất lịch theo tuần/công việc, sao chép.
- PWA, lưu cục bộ, đăng ký/đăng nhập Supabase, đồng bộ dữ liệu.

## Giới hạn

Chưa có thuật toán tự sinh lịch từ danh sách thời gian rảnh/ưu tiên. Hiện tại là quản lý và điều chỉnh lịch thủ công có kiểm tra quy tắc. Cần chạy SQL và bật Realtime trong Supabase trước khi dùng đồng bộ nhiều thiết bị. Chưa thực hiện kiểm thử end-to-end trên trình duyệt.
