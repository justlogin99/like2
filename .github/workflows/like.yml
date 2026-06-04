#!/usr/bin/env python3

import os, sys, time, traceback, random
from datetime import datetime, timezone, timedelta
from pathlib import Path
from typing import Optional, Dict, List, Any

import requests
from playwright.sync_api import sync_playwright, TimeoutError as PlaywrightTimeoutError

# ---------- 配置 ----------
API_BASE = "https://panel.godlike.host"
OUTPUT_DIR = Path("Godlike")
OUTPUT_DIR.mkdir(parents=True, exist_ok=True)
CN_TZ = timezone(timedelta(hours=8))

# ---------- 工具函数 ----------
def cn_time():
    return datetime.now(CN_TZ).strftime("%Y-%m-%d %H:%M:%S")

def mask_email(email: str) -> str:
    if not email or "@" not in email:
        return "***"
    user, domain = email.split("@", 1)
    return f"{user[:3]}***@{domain}"

def mask_server(server_id: str) -> str:
    if not server_id or len(server_id) < 6:
        return "***"
    return f"{server_id[:3]}***{server_id[-3:]}"

def snapshot(name: str) -> str:
    return str(OUTPUT_DIR / f"{name}_{int(time.time())}.png")

def notify_tg(ok: bool, email: str = "", server: str = "",
              before: str = "", after: str = "",
              error_msg: str = "", screenshot: str = None):
    token = os.environ.get("TG_BOT_TOKEN")
    chat_id = os.environ.get("TG_CHAT_ID")
    if not token or not chat_id:
        return

    msg = "✅ 续期成功\n\n" if ok else "❌ 续期失败\n\n"
    if email:   msg += f"账号：{email}\n"
    if server:  msg += f"服务器：{server}\n"
    if ok:
        if before and after:
            msg += f"到期：{before} → {after}\n"
        elif after:
            msg += f"到期：{after}\n"
        elif before:
            msg += f"到期：{before}\n"
    else:
        if error_msg: msg += f"原因：{error_msg}\n"
        if before:    msg += f"上次到期：{before}\n"
        if after:     msg += f"现在到期：{after}\n"
    msg += "\nGodlike Host Auto Renew"

    try:
        if screenshot and Path(screenshot).exists():
            with open(screenshot, "rb") as f:
                requests.post(
                    f"https://api.telegram.org/bot{token}/sendPhoto",
                    data={"chat_id": chat_id, "caption": msg},
                    files={"photo": f}, timeout=30)
        else:
            requests.post(
                f"https://api.telegram.org/bot{token}/sendMessage",
                json={"chat_id": chat_id, "text": msg, "disable_web_page_preview": True},
                timeout=30)
        print("[INFO] TG 通知已发送", flush=True)
    except Exception as e:
        print(f"[WARN] TG 通知发送失败: {e}", flush=True)

# ---------- Secret 与 Cookie 处理 ----------
def parse_raw_cookies(raw_cookie_str: str) -> List[Dict]:
    """解析原生 Cookie 字符串，仅提取登录必需的核心 Cookie"""
    if not raw_cookie_str:
        return []
    
    # 防止用户不小心复制了 "Cookie: " 请求头前缀
    if raw_cookie_str.lower().startswith("cookie: "):
        raw_cookie_str = raw_cookie_str[8:]
        
    valid_cookies = []
    
    for item in raw_cookie_str.split(';'):
        item = item.strip()
        if not item or '=' not in item:
            continue
            
        key, value = item.split('=', 1)
        key = key.strip()
        value = value.strip()
        
        # 仅保留 Pterodactyl 面板核心的会话/认证 Cookie
        if key in ["pterodactyl_session", "XSRF-TOKEN"] or key.startswith("remember_web_"):
            valid_cookies.append({
                "name": key,
                "value": value,
                "domain": "panel.godlike.host",
                "path": "/",
                "secure": True
            })
            
    return valid_cookies

def parse_secret(raw: str) -> Dict[str, Any]:
    parts = raw.strip().split("-----")
    if len(parts) < 2:
        raise ValueError("格式错误：必须提供账号和 Cookie 数据 (格式: 账号-----完整的Cookie字符串)")
    
    user = parts[0].strip()
    # 剩下的部分全部拼接为 Cookie 字符串，防止 Cookie 本身包含 "-----" 导致被错误截断
    raw_cookie = "-----".join(parts[1:]).strip()
    
    cookies = parse_raw_cookies(raw_cookie)
    if not cookies:
        print("[WARN] 警告：未能从提供的字符串中提取到核心登录 Cookie，请检查内容。", flush=True)
            
    return {"user": user, "cookies": cookies}

# ---------- API 交互 ----------
def api_get_servers(session: requests.Session) -> Optional[List[Dict]]:
    try:
        r = session.get(f"{API_BASE}/api/client",
                        params={"page": 1, "sort": "creation", "asc": "true"},
                        headers={"Accept": "application/json", "X-Requested-With": "XMLHttpRequest"},
                        timeout=30)
        r.raise_for_status()
        return r.json().get("data", [])
    except Exception as e:
        print(f"[ERROR] 获取服务器列表失败: {e}", flush=True)
        return None

def find_free_server(servers: List[Dict]) -> Optional[Dict]:
    for srv in servers:
        if srv["attributes"].get("free"):
            return srv
    return None

def get_free_timer(session: requests.Session, uuid: str) -> Optional[str]:
    servers = api_get_servers(session)
    if servers:
        for srv in servers:
            if srv["attributes"]["uuid"] == uuid:
                return srv["attributes"]["free_timer"]
    return None

def calc_remaining(timer: str) -> str:
    if not timer:
        return "未知"
    try:
        expire = datetime.fromisoformat(timer.replace("Z", "+00:00"))
        delta = expire - datetime.now(timezone.utc)
        secs = int(delta.total_seconds())
        if secs <= 0:
            return "已过期"
        d, rem = divmod(secs, 86400)
        h, rem = divmod(rem, 3600)
        m = rem // 60
        parts = []
        if d: parts.append(f"{d}天")
        if h: parts.append(f"{h}小时")
        if m: parts.append(f"{m}分钟")
        return " ".join(parts) if parts else "<1分钟"
    except:
        return timer

# ---------- Cookie 辅助 ----------
def session_from_cookies(cookie_list: List[Dict]) -> requests.Session:
    s = requests.Session()
    for c in cookie_list:
        s.cookies.set(
            c.get("name"), c.get("value"),
            domain=c.get("domain", "panel.godlike.host"),
            path=c.get("path", "/"),
        )
    return s

def test_cookie_valid(session: requests.Session) -> bool:
    try:
        r = session.get(f"{API_BASE}/api/client",
                        params={"page": 1},
                        headers={"Accept": "application/json"},
                        timeout=15)
        return r.status_code == 200 and "data" in r.json()
    except:
        return False

def safe_add_cookies(page, cookies):
    if cookies:
        page.context.add_cookies(cookies)
        print(f"[INFO] 成功注入 {len(cookies)} 个核心 Cookie", flush=True)
        print("[INFO] 刷新页面验证登录状态...", flush=True)
        page.goto(API_BASE, wait_until="domcontentloaded")
        page.wait_for_timeout(3000)

# ---------- 续期操作 ----------
def do_renewal(page, server_short_id: str, max_retries: int = 3) -> bool:
    url = f"{API_BASE}/server/{server_short_id}"
    for attempt in range(1, max_retries + 1):
        try:
            if attempt == 1:
                print("[INFO] 访问服务器页面...", flush=True)
                page.goto(url, wait_until="domcontentloaded")
            else:
                print(f"[INFO] 第{attempt}次重试，刷新页面...", flush=True)
                page.reload(wait_until="domcontentloaded")

            page.wait_for_timeout(5000)

            # 检查是否仍然有前端错误
            error_selectors = [
                'text="An error was encountered"',
                'text="error was encountered"',
                'text="Try refreshing the page"'
            ]
            has_error = False
            for sel in error_selectors:
                loc = page.locator(sel)
                if loc.count() > 0:
                    print(f"[WARN] 检测到页面错误: {sel}，将重试...", flush=True)
                    has_error = True
                    break

            if has_error and attempt < max_retries:
                continue
            elif has_error and attempt == max_retries:
                print("[ERROR] 多次重试后页面仍存在错误", flush=True)
                page.screenshot(path=snapshot("renewal_page_error"))
                return False

            # 等待续期按钮
            add_btn = page.locator('button:has-text("Add 90 minutes")')
            add_btn.wait_for(state="visible", timeout=30000)
            add_btn.click()
            print("[INFO] 已点击 Add 90 minutes", flush=True)

            ad_btn = page.locator('button:has-text("Watch advertisment")')
            ad_btn.wait_for(state="visible", timeout=10000)
            ad_btn.click()
            print("[INFO] 已点击 Watch advertisment", flush=True)

            print("[INFO] 等待广告 120 秒...", flush=True)
            time.sleep(120)

            # 广告结束后刷新页面
            page.reload(wait_until="domcontentloaded")
            page.wait_for_timeout(3000)
            return True

        except PlaywrightTimeoutError:
            print(f"[ERROR] 第{attempt}次：续期按钮未出现", flush=True)
            if attempt < max_retries:
                continue
            page.screenshot(path=snapshot("renewal_not_found"))
            return False
        except Exception as e:
            print(f"[ERROR] 续期异常 (重试{attempt}): {e}", flush=True)
            if attempt < max_retries:
                continue
            page.screenshot(path=snapshot("renewal_error"))
            return False

    return False

# ---------- 单账号流程 ----------
def process_account(key: str, proxy: str = None) -> bool:
    raw = os.environ.get(key, "").strip()
    if not raw:
        return True

    try:
        sec = parse_secret(raw)
    except Exception as e:
        print(f"[ERROR] {key} 格式错误: {e}", flush=True)
        notify_tg(False, email="", error_msg=str(e))
        return False

    user = sec["user"]
    cookielist = sec["cookies"]
    display_user = mask_email(user)

    print(f"\n{'='*60}\n[INFO] 处理 {key} ({display_user})\n{'='*60}", flush=True)

    if not cookielist:
        print("[ERROR] 未提取到有效的核心 Cookie", flush=True)
        notify_tg(False, email=user, error_msg="提取的 Cookie 无效，请检查提供的字符串")
        return False

    # 验证 Cookie
    session = session_from_cookies(cookielist)
    if test_cookie_valid(session):
        print("[INFO] 🍪 Cookie 登录验证成功", flush=True)
    else:
        print("[ERROR] Cookie 已失效，无法登录", flush=True)
        notify_tg(False, email=user, error_msg="Cookie 已失效，请手动更新环境变量")
        return False

    # 获取服务器
    servers = api_get_servers(session)
    if not servers:
        notify_tg(False, email=user, error_msg="无法获取服务器列表")
        return False
    srv = find_free_server(servers)
    if not srv:
        notify_tg(False, email=user, error_msg="未找到免费服务器")
        return False

    uuid = srv["attributes"]["uuid"]
    short_id = srv["attributes"]["identifier"]
    before = calc_remaining(srv["attributes"].get("free_timer"))
    print(f"服务器: {mask_server(uuid)}, 续期前剩余: {before}", flush=True)

    if before == "已过期":
        with sync_playwright() as p:
            browser = p.chromium.launch(headless=True, proxy={"server": proxy} if proxy else None)
            page = browser.new_page()
            safe_add_cookies(page, cookielist)
            page.goto(f"{API_BASE}/server/{short_id}", wait_until="domcontentloaded")
            page.wait_for_timeout(3000)
            ss_path = snapshot("expired")
            page.screenshot(path=ss_path)
            browser.close()
        notify_tg(False, email=user, server=short_id, before=before,
                  error_msg="服务器已过期，无法续期", screenshot=ss_path)
        return False

    # 续期
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True, proxy={"server": proxy} if proxy else None)
        page = browser.new_page()
        success_ss = None
        try:
            safe_add_cookies(page, cookielist)
            if not do_renewal(page, short_id):
                fail_ss = snapshot("renewal_fail")
                page.screenshot(path=fail_ss)
                notify_tg(False, email=user, server=short_id, before=before,
                          error_msg="续期点击失败", screenshot=fail_ss)
                return False

            after = calc_remaining(get_free_timer(session, uuid))
            print(f"[INFO] 续期后剩余: {after}", flush=True)

            success_ss = snapshot("success")
            page.screenshot(path=success_ss)

            notify_tg(True, email=user, server=short_id, before=before, after=after,
                      screenshot=success_ss)
            print(f"[INFO] ✅ {key} 续期成功", flush=True)
            return True

        except Exception as e:
            print(f"[ERROR] 续期流程异常: {e}", flush=True)
            traceback.print_exc()
            exc_ss = snapshot("exception")
            try:
                page.screenshot(path=exc_ss)
            except:
                pass
            notify_tg(False, email=user, server=short_id, before=before,
                      error_msg=f"脚本异常: {str(e)[:200]}", screenshot=exc_ss)
            return False
        finally:
            if success_ss is None:
                try:
                    page.screenshot(path=snapshot("final_error"))
                except:
                    pass
            browser.close()

def main():
    proxy = os.environ.get("PROXY_SERVER", "")
    if proxy:
        print(f"[INFO] 代理: {proxy}", flush=True)

    accounts = [f"GODLIKE_{i}" for i in range(1, 6)]
    all_ok = True
    for idx, acc in enumerate(accounts):
        try:
            ok = process_account(acc, proxy if proxy else None)
            if not ok:
                all_ok = False
        except Exception as e:
            print(f"[FATAL] {acc} 崩溃: {e}", flush=True)
            traceback.print_exc()
            user = ""
            try:
                raw = os.environ.get(acc, "")
                if raw:
                    parts = raw.split("-----")
                    if parts: user = parts[0].strip()
            except: pass
            notify_tg(False, email=user, error_msg=f"脚本异常: {str(e)[:200]}")
            all_ok = False
        if idx < len(accounts) - 1:
            time.sleep(random.randint(5, 15))

    if all_ok:
        print("[INFO] 🎉 所有账号处理成功", flush=True)
        sys.exit(0)
    else:
        print("[ERROR] 部分账号处理失败", flush=True)
        sys.exit(1)

if __name__ == "__main__":
    main()
