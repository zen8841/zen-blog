---
title: OneTab v2 資料遺失與遷移 NiceTab 記錄
katex: false
mathjax: false
mermaid: false
categories:
  - 雜七雜八
tags:
  - OneTab
  - NiceTab
excerpt: 關於 OneTab v2 的資料存放與如何搶救記錄並轉移到 NiceTab
date: 2026-06-30 23:13:56
updated: 2026-06-30 23:13:56
index_img:
banner_img:
---


{% note danger%}

如果你正在使用 OneTab v2 最好將瀏覽器的`Default/IndexedDB/chrome-extension_chphlpgkkbolifaimnlloiipkdnihall_0.indexeddb.leveldb`定時備份， v2 不再使用`Default/Local Extension Settings/chphlpgkkbolifaimnlloiipkdnihall`儲存資料，只用這個目錄無法恢復或是 dump 出資料

{% endnote %}

# 前言

其實我並不是不知道 OneTab 會丟資料這回事，在 v1 時代我就已經丟過兩次了，第一次剛好在不久前有 export 過，也沒損失多少資料，第二次就學乖了，有同步瀏覽器的`Default/Local Extension Settings/chphlpgkkbolifaimnlloiipkdnihall`資料夾，也沒丟失任何資料。在這次之後就從 OneTab 換到 Toby 使用，直到 Toby 開始收費才換回 Linkwarden + OneTab

# 意外

這次丟資料一開始也沒覺得有什麼，但是在覆蓋了`Local Extension Settings`資料夾沒有任何反應就開始有點慌張了，不過有查到[一篇文章](https://6bcf7279.info/2023/01/12/24750770/)[^1]教怎麼從 OneTab 的 LevelDB 中 dump 資料出來，嘗試過後的確成功把資料 dump 出來了，但壞消息是，只剩去年10月底以前的資料，正好就是 OneTab 升級 v2 的時間。

在經過一些測試之後發現 OneTab 升級到 v2 後改為使用 IndexedDB 儲存資料，但是這件事完全沒有任何通知，沒有任何 changelog 或其他通知，在明知道自己的產品很容易發生資料遺失(可以去看評論/ reddit 很多人抱怨)的情況下還不通知使用者備份，也是因為這個原因我目前改為使用 NiceTab。

# 轉移

雖然資料只剩到去年10月的，但也不能因此放棄，還是要想辦法轉移出來，在文章中[^1]提到的兩種 dump 工具都會匯出出相同的資料，不過如果把 LevelDB 中的`.log`檔案改成`.ldb`還可以額外 dump 出一些資料。

關於整理這些 JSON data 的腳本我就懶得自己寫了，這個時代就直接丟給 AI 然後小幅度修正/檢查就好

{% fold info @附上 AI 生出來的爛 code %}

merge 不同方式 dump 出來的 data

```python
import json
from collections import OrderedDict

SRC_DIR = 'oridumpdata'
OUT_DIR = 'tabdata'

src_cfgs = [
    ('179log.json', 'stateMigratedToIDB', False),
    ('179ldb.json', 'state', False),
    ('pydumper.json', 'stateMigratedToIDB', False),
    ('LevelDBDumper.json', 'stateMigratedToIDB', True),
]

def load_source(sname, key, double_unescape):
    with open(f'{SRC_DIR}/{sname}') as f:
        d = json.load(f)
    if double_unescape:
        raw = d[0]['data'][key]
        raw = json.loads(raw)
    else:
        raw = d[key]
    return json.loads(raw)

all_sources = [cfg[0] for cfg in src_cfgs]

groups_merged = OrderedDict()

for sname, key, double in src_cfgs:
    data = load_source(sname, key, double)
    for g in data['tabGroups']:
        gid = g['id']
        if gid not in groups_merged:
            groups_merged[gid] = {
                'id': gid,
                'createDate': g['createDate'],
                'label': g.get('label'),
                'sources': set(),
                'tabs': OrderedDict(),
            }
        gm = groups_merged[gid]
        gm['sources'].add(sname)
        for t in g['tabsMeta']:
            url = t['url']
            if url not in gm['tabs']:
                gm['tabs'][url] = {
                    'url': url,
                    'title': t['title'],
                    'sources': set(),
                }
            gm['tabs'][url]['sources'].add(sname)

other3 = {'179log.json', 'pydumper.json', 'LevelDBDumper.json'}

merged_output = []
diff_ldb_only = []
diff_ldb_missing = []
diff_others = []

for gid, gm in groups_merged.items():
    merged_tabs = []
    tabs_ldb_only = []
    tabs_ldb_missing = []
    tabs_others = []
    for url, tab in gm['tabs'].items():
        src_list = sorted(tab['sources'])
        merged_tabs.append({
            'url': url,
            'title': tab['title'],
            'sources': src_list,
        })
        ss = tab['sources']
        if ss == {'179ldb.json'}:
            tabs_ldb_only.append({
                'url': url,
                'title': tab['title'],
                'present_in': sorted(ss),
                'missing_from': sorted(other3),
            })
        elif '179ldb.json' not in ss:
            missing = [s for s in all_sources if s not in ss]
            tabs_ldb_missing.append({
                'url': url,
                'title': tab['title'],
                'present_in': sorted(ss),
                'missing_from': missing,
            })
        elif ss & other3 and len(ss & other3) < 3:
            present = sorted(ss & other3)
            missing = sorted(other3 - ss)
            tabs_others.append({
                'url': url,
                'title': tab['title'],
                'present_in_others': present,
                'missing_from_others': missing,
                'also_in_ldb': '179ldb.json' in ss,
            })
    merged_output.append({
        'id': gid,
        'createDate': gm['createDate'],
        'label': gm['label'],
        'tabs': merged_tabs,
    })
    def maybe_append(dst, tabs):
        if tabs:
            dst.append({
                'id': gid, 'createDate': gm['createDate'],
                'label': gm['label'], 'tabs': tabs,
            })
    maybe_append(diff_ldb_only, tabs_ldb_only)
    maybe_append(diff_ldb_missing, tabs_ldb_missing)
    maybe_append(diff_others, tabs_others)

# Also output per-source clean tab files
for sname, key, double in src_cfgs:
    data = load_source(sname, key, double)
    base = sname.replace('.json', '')
    base = base.replace('LevelDBDumper', 'levaldumper')
    outname = f'{base}-tab.json'
    with open(f'{OUT_DIR}/{outname}', 'w', encoding='utf-8') as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

with open(f'{OUT_DIR}/merged.json', 'w', encoding='utf-8') as f:
    json.dump(merged_output, f, ensure_ascii=False, indent=2)
with open(f'{OUT_DIR}/diff-179ldb-only.json', 'w', encoding='utf-8') as f:
    json.dump(diff_ldb_only, f, ensure_ascii=False, indent=2)
with open(f'{OUT_DIR}/diff-179ldb-missing.json', 'w', encoding='utf-8') as f:
    json.dump(diff_ldb_missing, f, ensure_ascii=False, indent=2)
with open(f'{OUT_DIR}/diff-others.json', 'w', encoding='utf-8') as f:
    json.dump(diff_others, f, ensure_ascii=False, indent=2)

total_tabs = sum(len(g['tabs']) for g in merged_output)
n1 = sum(len(g['tabs']) for g in diff_ldb_only)
n2 = sum(len(g['tabs']) for g in diff_ldb_missing)
n3 = sum(len(g['tabs']) for g in diff_others)
print(f"merged:              {len(merged_output)} groups, {total_tabs} tabs")
print(f"diff-179ldb-only:    {len(diff_ldb_only)} groups, {n1} tabs")
print(f"diff-179ldb-missing: {len(diff_ldb_missing)} groups, {n2} tabs")
print(f"diff-others:         {len(diff_others)} groups, {n3} tabs")
print("done")
```

把 merge 好的 data 轉成 NiceTab 的 Import 格式

```python
import json
from datetime import datetime

OTHER3 = {'179log.json', 'pydumper.json', 'LevelDBDumper.json'}
ALL4 = OTHER3 | {'179ldb.json'}

def source_label(sources):
    ss = set(sources)
    if ss == ALL4:
        return 'both'
    if ss == {'179ldb.json'}:
        return '179ldbonly'
    if ss == OTHER3:
        return '179ldbmissing'
    return 'unknown'

def ms_to_dt(ms):
    return datetime.fromtimestamp(ms / 1000).strftime('%Y-%m-%d %H:%M:%S')

with open('tabdata/merged.json') as f:
    merged = json.load(f)

group_list = []
for g in merged:
    gid = g['id']
    label = g.get('label')
    create_date = g.get('createDate')
    tabs = g['tabs']

    labels_per_tab = [source_label(t['sources']) for t in tabs]
    unique_labels = set(labels_per_tab)

    if len(unique_labels) == 1:
        sl = unique_labels.pop()
        base_name = label if label else f'untitled-{gid[:8]}'
        group_name = f'{base_name} [{sl}]' if sl != 'both' else base_name
        tab_list = [{'title': t['title'], 'url': t['url']} for t in tabs]
    else:
        base_name = label if label else f'untitled-{gid[:8]}'
        group_name = base_name
        tab_list = []
        for t in tabs:
            sl = source_label(t['sources'])
            title = t['title']
            if sl != 'both':
                title = f'{title} [{sl}]'
            tab_list.append({'title': title, 'url': t['url']})

    group_list.append({
        'createTime': ms_to_dt(create_date) if create_date else '1970-01-01 00:00:00',
        'groupName': group_name,
        'isLocked': False,
        'isStarred': False,
        'tabList': tab_list,
    })

output = [{
    'createTime': datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
    'groupList': group_list,
}]

with open('onetab-to-nicetab.json', 'w', encoding='utf-8') as f:
    json.dump(output, f, ensure_ascii=False, indent=2)

total_tabs = sum(len(g['tabList']) for g in group_list)
n_unknown = sum(1 for g in merged for t in g['tabs'] if source_label(t['sources']) == 'unknown')
print(f"onetab-to-nicetab.json: {len(group_list)} groups, {total_tabs} tabs")
print(f"unknown source tags: {n_unknown}")
print("done")
```

{% endfold %}

差不多就是這樣，還是建議看到這裡的人如果有在用 OneTab，最好還是找個替代選擇換了吧，全 loss data 真的蠻痛的。

## 參考

[^1]: [搶救 OneTab 資料 Save my OneTab data | ](https://6bcf7279.info/2023/01/12/24750770/)
