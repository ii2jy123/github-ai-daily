import datetime
import json
import os
import time
import urllib.error
import urllib.parse
import urllib.request
from pathlib import Path

TZ = datetime.timezone(datetime.timedelta(hours=8))
TODAY = datetime.datetime.now(TZ).date()
SINCE = TODAY - datetime.timedelta(days=14)
TOPICS = {
    'AI Agent': 'topic:ai-agent OR topic:agentic-ai',
    'AI自动化': 'topic:automation OR topic:ai-automation',
    '个人效率工具': 'topic:productivity OR topic:personal-assistant',
    'AI内容创作': 'topic:generative-ai OR topic:ai-content-generation',
    '全部热门项目': 'stars:>1000',
    '开源知识管理': 'topic:knowledge-management OR topic:second-brain',
    'AI编程/Codex': 'topic:coding-agent OR topic:ai-coding-assistant',
    '一人公司创业工具': 'topic:indie-hacker OR topic:solopreneur',
}

def get_json(url, token=None):
    headers = {'User-Agent': 'github-ai-daily', 'Accept': 'application/vnd.github+json'}
    if token:
        headers['Authorization'] = f'Bearer {token}'
    req = urllib.request.Request(url, headers=headers)
    with urllib.request.urlopen(req, timeout=30) as res:
        return json.loads(res.read().decode('utf-8'))

def main():
    token = os.getenv('GH_SEARCH_TOKEN')
    candidates = {}
    errors = []
    for label, query in TOPICS.items():
        # Search each topic separately; GitHub's search parser does not accept grouped OR clauses reliably.
        for term in query.split(' OR '):
            q = f'{term} pushed:>={SINCE} archived:false fork:false'
            url = 'https://api.github.com/search/repositories?' + urllib.parse.urlencode({
                'q': q, 'sort': 'stars', 'order': 'desc', 'per_page': 12
            })
            try:
                for item in get_json(url, token).get('items', []):
                    name = item['full_name']
                    if name not in candidates:
                        candidates[name] = {'item': item, 'topics': set()}
                    candidates[name]['topics'].add(label)
            except (urllib.error.HTTPError, urllib.error.URLError, TimeoutError) as e:
                errors.append(f'{label}: {e}')
            time.sleep(0.5)
    if not candidates:
        raise RuntimeError('GitHub 搜索失败：' + '; '.join(errors))

    history_file = Path('data/history.json')
    try:
        history = json.loads(history_file.read_text(encoding='utf-8'))
    except (FileNotFoundError, json.JSONDecodeError):
        history = []
    recent = set(history[-100:])
    ranked = sorted(candidates.items(), key=lambda pair: (
        pair[0] not in recent,
        len(pair[1]['topics']),
        pair[1]['item'].get('stargazers_count', 0)
    ), reverse=True)
    selected = ranked[:10]
    lines = [f'# GitHub 每日精选 · {TODAY}', '',
             '覆盖 AI Agent、自动化、效率工具、内容创作、热门项目、知识管理、AI 编程、一人公司。',
             '说明：根据主题匹配、近期活跃度、总 Star 和历史推荐记录筛选；并非每日涨星排行榜。', '']
    for index, (name, entry) in enumerate(selected, 1):
        item = entry['item']
        desc = (item.get('description') or '暂无项目简介').replace('\n', ' ')[:180]
        lines += [f'**{index}. {name}**',
                  f'分类：{"、".join(sorted(entry["topics"]))}｜⭐ {item.get("stargazers_count", 0):,}｜语言：{item.get("language") or "未标注"}',
                  f'简介（原文）：{desc}',
                  f'地址：{item["html_url"]}', '']
    if errors:
        lines += [f'提示：有 {len(errors)} 次分类搜索失败，结果可能不完整。', '']
    report = '\n'.join(lines)
    Path('reports').mkdir(exist_ok=True)
    Path('reports/latest.md').write_text(report, encoding='utf-8')
    print(f'已生成 {len(selected)} 个项目的日报')

    sendkey = os.getenv('SERVERCHAN_SENDKEY', '').strip()
    if not sendkey:
        raise RuntimeError('缺少 SERVERCHAN_SENDKEY，请在仓库 Actions Secrets 中配置。')
    body = urllib.parse.urlencode({'title': f'GitHub每日精选 {TODAY}', 'desp': report}).encode()
    req = urllib.request.Request(
        'https://sctapi.ftqq.com/' + urllib.parse.quote(sendkey, safe='') + '.send',
        data=body,
        headers={'Content-Type': 'application/x-www-form-urlencoded', 'User-Agent': 'github-ai-daily'},
        method='POST'
    )
    with urllib.request.urlopen(req, timeout=30) as res:
        result = json.loads(res.read().decode('utf-8'))
    if str(result.get('code')) != '0':
        raise RuntimeError('Server酱未接受推送：' + str(result.get('message', result.get('code'))))
    print('Server酱已接受推送请求，请检查微信是否收到')
    history.extend([name for name, _ in selected])
    history_file.parent.mkdir(exist_ok=True)
    history_file.write_text(json.dumps(history[-200:], ensure_ascii=False, indent=2), encoding='utf-8')

if __name__ == '__main__':
    main()
