---
title: "Astro 自动生成文章摘要"
date: "2025-07-22 22:01:14"
pubDatetime: 2025-07-22 22:01:14
description: "介绍了 Astro 博客怎么自动生成文章摘要。"
tags: ["博客搭建记录"]
---
像我这种懒人，很讨厌写摘要啊介绍啊之类的，而 SEO 又必须写，所以只好写一个 Python 脚本去完成这个事情了：
```python
#!/usr/bin/env python3
import os
import re
import time
import argparse
import requests
from datetime import datetime, timedelta

# API 配置
API_URL = "https://api.siliconflow.cn/v1/chat/completions"
API_MODEL = "Qwen/Qwen2-7B-Instruct"
REQUEST_DELAY = 0.2  # 300ms 请求间隔
MAX_RETRIES = 3  # 失败重试次数


class DescriptionRewriter:
    def __init__(self):
        self.processed = 0
        self.skipped = 0
        self.failed = 0
        self.start_time = datetime.now()
        self.total_tokens = 0
        self.rate_limit_hit = False
        self.rate_limit_reset = None
        self.api_key = os.getenv("SILICONFLOW_API_KEY")

        if not self.api_key:
            print("❌ 错误: 请设置环境变量 SILICONFLOW_API_KEY")
            exit(1)

    def rewrite_description(self, content):
        """使用AI模型重写描述内容"""
        prompt = (
                "请根据以下文章内容，重新编写一个简洁、吸引人的description（描述），"
                "要求：\n"
                "1. 长度在30-50字之间\n"
                "2. 保持专业和技术性\n"
                "3. 突出技术要点\n"
                "4. 避免使用引号（单双都是）\n"
                "5. 不要出现网站链接\n"
                "6. 所有内容都要在同一行\n"
                "6. 避免出现例如列表等内容\n"
                "7. 你要站在读者的口吻上，不要出现类似于：简化版本的描述、我们通过简化描述得到了... 等等诸如此类的话，不要出现AI的痕迹\n"
                "原始内容：\n" + content
        )

        payload = {
            "model": API_MODEL,
            "messages": [{"role": "user", "content": prompt}]
        }

        headers = {
            "Authorization": f"Bearer {self.api_key}",
            "Content-Type": "application/json"
        }

        for attempt in range(MAX_RETRIES):
            try:
                response = requests.post(
                    API_URL,
                    json=payload,
                    headers=headers,
                    timeout=30
                )
                response.raise_for_status()

                # 解析响应
                data = response.json()
                new_desc = data["choices"][0]["message"]["content"].strip()
                usage = data.get("usage", {})

                # 统计token使用
                self.total_tokens += usage.get("total_tokens", 0)

                # 检查速率限制
                if "x-ratelimit-remaining" in response.headers:
                    remaining = int(response.headers["x-ratelimit-remaining"])
                    if remaining < 10:  # 接近限额时警告
                        print(f"⚠️ 警告: 剩余API额度仅剩 {remaining}")

                    if remaining == 0:
                        reset_time = int(response.headers.get("x-ratelimit-reset", 60))
                        self.rate_limit_hit = True
                        self.rate_limit_reset = datetime.now() + timedelta(seconds=reset_time)
                        return None

                return new_desc

            except requests.exceptions.HTTPError as e:
                if response.status_code == 429:  # 速率限制错误
                    reset_time = int(response.headers.get("x-ratelimit-reset", 60))
                    self.rate_limit_hit = True
                    self.rate_limit_reset = datetime.now() + timedelta(seconds=reset_time)
                    print(f"⏳ API限额已达，将在 {reset_time} 秒后重置")
                    return None
                print(f"API请求错误: {e} (尝试 {attempt + 1}/{MAX_RETRIES})")

            except Exception as e:
                print(f"API请求失败: {str(e)} (尝试 {attempt + 1}/{MAX_RETRIES})")

            time.sleep(2 ** attempt)  # 指数退避

        return None

    def process_file(self, file_path):
        """处理单个Markdown文件"""
        try:
            with open(file_path, "r+", encoding="utf-8") as f:
                content = f.read()

                # 匹配front matter中的description字段
                pattern = r'^---\s*\n(.*?description:\s*")(.*?)("\s*\n.*?)\n---'
                match = re.search(pattern, content, re.DOTALL)

                if not match:
                    print(f"⚠️ {file_path} 未找到description字段，跳过")
                    self.skipped += 1
                    return False

                prefix = match.group(1)
                old_desc = match.group(2)
                suffix = match.group(3)

                # 获取新的描述内容
                new_desc = self.rewrite_description(old_desc)

                if not new_desc:
                    print(f"❌ {file_path} 未能获取新描述，跳过")
                    self.failed += 1
                    return False

                # 替换内容
                new_content = content.replace(
                    f'{prefix}{old_desc}{suffix}',
                    f'{prefix}{new_desc}{suffix}',
                    1
                )

                # 写回文件
                f.seek(0)
                f.write(new_content)
                f.truncate()

                print(f"✅ {file_path} 描述已更新")
                print(f"  原描述: {old_desc[:60]}...")
                print(f"  新描述: {new_desc[:60]}...\n")
                self.processed += 1
                return True

        except Exception as e:
            print(f"❌ 处理 {file_path} 时出错: {str(e)}")
            self.failed += 1
            return False

    def process_directory(self, directory):
        """处理目录中的所有Markdown文件"""
        md_files = []
        for root, _, files in os.walk(directory):
            for file in files:
                if file.lower().endswith(".md"):
                    md_files.append(os.path.join(root, file))

        if not md_files:
            print(f"❌ 在 {directory} 中未找到Markdown文件")
            return

        total_files = len(md_files)
        print(f"📂 找到 {total_files} 个Markdown文件")
        print(f"⏱️ 预计处理时间: {timedelta(seconds=total_files * REQUEST_DELAY)}")

        for i, file_path in enumerate(md_files):
            if self.rate_limit_hit:
                wait_seconds = (self.rate_limit_reset - datetime.now()).total_seconds()
                if wait_seconds > 0:
                    print(f"⏳ 达到API限额，等待 {wait_seconds:.0f} 秒...")
                    time.sleep(wait_seconds)
                self.rate_limit_hit = False

            print(f"\n📄 处理文件 {i + 1}/{total_files}: {os.path.basename(file_path)}")
            self.process_file(file_path)

            # 请求间隔控制
            if i < total_files - 1:
                time.sleep(REQUEST_DELAY)

    def print_summary(self):
        """打印处理摘要"""
        duration = datetime.now() - self.start_time
        print("\n" + "=" * 50)
        print("📊 处理摘要")
        print("=" * 50)
        print(f"✅ 成功处理: {self.processed} 个文件")
        print(f"⚠️ 跳过处理: {self.skipped} 个文件")
        print(f"❌ 处理失败: {self.failed} 个文件")
        print(f"⏱️ 总耗时: {duration}")
        print(f"🧠 总消耗Token: {self.total_tokens}")
        print(f"🚀 平均速度: {self.processed / duration.total_seconds() * 60:.1f} 文件/分钟")
        print("=" * 50)


def main():
    parser = argparse.ArgumentParser(
        description="Markdown描述智能重写工具",
        formatter_class=argparse.ArgumentDefaultsHelpFormatter
    )
    parser.add_argument(
        "directory",
        help="包含Markdown文件的目录路径"
    )
    args = parser.parse_args()

    if not os.path.isdir(args.directory):
        print(f"❌ 错误: {args.directory} 不是有效目录")
        return

    rewriter = DescriptionRewriter()
    rewriter.process_directory(args.directory)
    rewriter.print_summary()


if __name__ == "__main__":
    main()
```
使用的时候需要注意：设置环境变量：`export SILICONFLOW_API_KEY="xxxx"`，我这里用的是硅基流动，你可以根据你的API平台换成自己的。