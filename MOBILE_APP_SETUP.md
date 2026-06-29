# Wasabi 手机 App 与云同步配置

当前卡片程序已经升级为 PWA。完成一次云端配置和 HTTPS 发布后，可以：

- 在安卓 Chrome 中安装到桌面；
- 像普通 App 一样全屏使用；
- 离线复习已经加载过的内容；
- 使用同一邮箱账号同步手机与电脑的学习进度；
- 从云端获取每日新增卡片。

## 1. 创建 Supabase 项目

在 Supabase 新建一个项目，然后在 SQL Editor 中执行：

```sql
create table public.wasabi_cards (
  id text primary key,
  type text not null check (type in ('phrase', 'pattern', 'eudic')),
  front text not null,
  back text not null,
  example text not null default '',
  challenge text not null default '',
  updated_at timestamptz not null default now()
);

create table public.wasabi_progress (
  user_id uuid primary key references auth.users(id) on delete cascade,
  payload jsonb not null default '{}'::jsonb,
  updated_at timestamptz not null default now()
);

alter table public.wasabi_cards enable row level security;
alter table public.wasabi_progress enable row level security;

create policy "Authenticated users can read cards"
on public.wasabi_cards for select
to authenticated
using (true);

create policy "Users can read their own progress"
on public.wasabi_progress for select
to authenticated
using (auth.uid() = user_id);

create policy "Users can insert their own progress"
on public.wasabi_progress for insert
to authenticated
with check (auth.uid() = user_id);

create policy "Users can update their own progress"
on public.wasabi_progress for update
to authenticated
using (auth.uid() = user_id)
with check (auth.uid() = user_id);
```

在 Supabase Authentication 设置中启用 Email 登录。可以保留邮箱验证，也可以在个人项目中关闭验证以简化首次登录。

## 2. 配置 App

编辑 `wasabi-cloud-config.js`：

```js
window.WASABI_CLOUD = {
  supabaseUrl: "https://YOUR_PROJECT.supabase.co",
  supabaseAnonKey: "YOUR_ANON_KEY"
};
```

`anon key` 可以放在前端；不要把 `service_role key` 写进该文件。

在电脑环境变量中保存用于每日上传卡片的私密配置：

```text
SUPABASE_URL=https://YOUR_PROJECT.supabase.co
SUPABASE_SERVICE_ROLE_KEY=你的 service_role key
```

每日新增卡片后运行：

```powershell
node work\sync-cards-to-supabase.cjs
```

## 3. 发布为 HTTPS 网站

把 `outputs` 文件夹发布到 GitHub Pages、Cloudflare Pages、Netlify 或其他 HTTPS 静态网站。入口文件是：

```text
wasabi-cards.html
```

PWA 安装和 Service Worker 需要 HTTPS；直接打开 `file://` 文件不能安装为手机 App。

## 4. 安卓安装

1. 用安卓 Chrome 打开发布后的网址。
2. 点击浏览器菜单中的“安装应用”或“添加到主屏幕”。
3. 打开 Wasabi App，点击右上角云朵按钮。
4. 注册或登录邮箱账号。
5. 在电脑端打开同一个网址，并使用同一账号登录。

此后每次答题先写入本机，联网时自动与云端合并。不同设备修改同一张卡时，以该卡最近一次学习记录为准。
