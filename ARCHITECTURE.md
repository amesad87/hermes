# معماری فنی

## دیتابیس (Supabase Postgres — پروژه kcmcovspbznredsiyqva)

### جدول‌ها
- `bot_settings` (key/value) — نگه‌داری کلیدها: telegram_bot_token, gemini_api_key, channel_chat_id, channel2_chat_id, last_update_id
- `posted_news` — دیدوپ اخبار پست‌شده در کانال ۱ و ۲ (source_link یکتا) + image_url
- `news_run_log` — لاگ اجرای کرون‌جاب‌های کانال ۱
- `motivational_log` — تاریخچه‌ی پست‌های انگیزشی کانال ۱ (جلوگیری از تکرار مضمون)
- `channel2_ads` — لیست پروژه‌های تبلیغاتی (ربات‌ها/سایت‌های آریامیر) برای چرخش تصادفی
- `channel2_log` — لاگ اجرای کانال ۲ (نوع پست + متن)
- `chat_history` — حافظه‌ی مکالمه‌ی چت‌بات (per chat_id، آخرین ۱۰ پیام)
- `posted_videos` — (باقی‌مانده از ایده‌ی اولیه‌ی ربات یوتیوب pv، فعلاً بلااستفاده)

### توابع اصلی (PL/pgSQL، با extensions.http_get/http_post برای فراخوانی API خارجی)

**کانال ۱ (Ariamir_academy):**
- `fetch_fresh_news_item()` — از فیدهای TechCrunch AI / Search Engine Journal / Marketing Dive خبر تازه می‌گیره + og:image استخراج می‌کنه
- `generate_channel_post(title, desc, source)` — با Gemini، خبر رو در قالب دقیق کانال (تیتر+بولت+هشتگ) بازنویسی می‌کنه
- `run_hourly_channel_post()` — orchestrator: خبر می‌گیره → می‌نویسه → با عکس پست می‌کنه → دیدوپ ثبت می‌کنه
- `generate_motivational_post()` / `run_motivational_post()` — پست انگیزشی سنگین فارسی/انگلیسی، با عکس بازیافتی از اخبار اخیر

**کانال ۲ (ARIAMIRZONE):**
- `gemini_generate(prompt)` — تابع عمومی فراخوانی Gemini (استفاده‌شده در بیشتر generatorها)
- `gen_c2_fact/history/famous_iranian/emotional/ad/poll/song()` — تولیدکننده‌های هر نوع محتوا
- `run_channel2_post()` — dispatcher وزن‌دار تصادفی بین ۹ نوع محتوا

**زیرساخت مشترک یوتیوب (بدون نیاز به API Key):**
- `yt_channel_videos(handle)` / `yt_search_videos(query)` — پارس مستقیم HTML یوتیوب (ساختار jsonpath روی ytInitialData → lockupViewModel/videoRenderer)

**چت‌بات:**
- `poll_and_reply_chat()` — هر ۱ دقیقه، getUpdates تلگرام رو چک می‌کنه، به پیام‌های پی‌وی با Gemini (+ حافظه‌ی ۱۰ پیام اخیر) جواب می‌ده

### کرون‌جاب‌ها (pg_cron، UTC)
| نام | Schedule (UTC) | معادل تهران | کار |
|---|---|---|---|
| `hourly-channel-news-post` | `30 2-23 * * *` | هر ساعت، ۶ صبح-۳ شب | خبر کانال ۱ |
| `motivational-post-every-2h` | `15 3-23/2 * * *` | هر ۲ ساعت | انگیزشی کانال ۱ |
| `hourly-channel2-mixed-post` | `45 2-23 * * *` | هر ساعت | محتوای ترکیبی کانال ۲ |
| `ai-chat-poll` | `* * * * *` | هر ۱ دقیقه، ۲۴/۷ | چت‌بات پی‌وی |

### امنیت
RLS روی همه‌ی جدول‌های public فعاله (بدون policy عمومی) — یعنی anon key دیگه به هیچ‌کدوم دسترسی نداره؛
فقط نقش `postgres` (که pg_cron باهاش اجرا می‌شه) از RLS عبور می‌کنه.

### مدل هوش مصنوعی
`gemini-3.6-flash` (تیر رایگان Google AI Studio) — مصرف فعلی حدود ۴۴ درخواست/روز بین دو کانال + مصرف متغیر چت‌بات، در تیر رایگان (۲۵۰-۱۵۰۰ درخواست/روز بسته به مدل) کاملاً جا داره.
