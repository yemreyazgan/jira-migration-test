# 📊 Jira to GitHub Issue Migration Project

Bu depo, Jira üzerindeki görevlerin, iş tiplerinin, öncelik derecelerinin, alt görevlerin (subtask), efor puanlarının (Story Point) ve versiyon bilgilerinin GitHub Issues mimarisine kurumsal standartlarda taşınması (migration) amacıyla geliştirilmiştir.

---

## 🎯 Proje Yapısı ve Eşleşmeler

Jira'daki mevcut proje yönetim yapısı, GitHub mimarisine birebir ve dinamik olarak şu şekilde entegre edilmiştir:

* **Özgün Görev Kodları (Jira Keys):** Jira'daki her bir görev ve alt görev, özgün anahtar kodlarıyla (`MS-1`, `MS-2`, `MS-4` vb.) GitHub Issue başlıklarına birebir taşınmıştır:
  * Ana Task Formatı: `[MS-1] API Sorgularında Optimizasyon...`
  * Subtask Formatı: `[MS-4] [Subtask of MS-1] PostgreSQL tarafındaki 'users' tablosuna index eklenecek`
* **Proje Metadataları (Metadata Table):** Her Issue'nun en üstüne kurumsal formatta Markdown tablosu eklenmiştir:
  * **Jira Key:** Özgün görev kodu.
  * **Story Point:** Efor puanı.
  * **Fix Version & Affects Version:** Sürüm ve ortam bilgileri.
  * **Build Info:** Derleme/Build detayları.
  * **Due Date:** Bitiş/Teslim tarihi.
* **İş İçerikleri (Description):** Görevlerin teknik detayları ve açıklamaları hatasız bir şekilde biçimlendirilerek taşınmıştır.
* **Tıklanabilir Alt Görevler (Subtasks):** Ana görevlerin detay sayfalarında, o göreve bağlı olan subtask'lar Issue numaralarıyla birlikte (`- [ ] #130 - [MS-4] Alt Görev Metni`) etkileşimli ve tıklanabilir bağlantılar şeklinde listelenmiştir.
* **Sorumlu Atamaları (Assignee):** Taşınan tüm görevler, projenin takibi için ilgili geliştiriciye (`yemreyazgan`) otomatik olarak atanmıştır.

---

## 🏷️ Kurumsal Etiket (Label) Standartları

Sürecin daha şeffaf ve izlenebilir olması için görevler dinamik ve renkli etiketlerle kategorize edilmiştir:

### 1. Öncelik Seviyeleri (Priority)
* `Priority: High` (Kırmızı) -> Kritik ve öncelikli çözülmesi gereken işler.
* `Priority: Medium` (Sarı) -> Standart geliştirme süreçleri.
* `Priority: Low` (Yeşil) -> Düşük öncelikli işler.

### 2. İş Tipleri (Issue Type)
* `Type: Story` (Mavi) -> Kullanıcı hikayeleri ve genel iş gereksinimleri.
* `Type: Task` (Mor) -> Teknik geliştirmeler ve operasyonel işler.
* `Type: Bug` (Koyu Kırmızı) -> Hata düzeltmeleri.
* `Type: Subtask` (Yeşil) -> Alt görev bileşenleri.

### 3. Parent ve Efor Bağlantıları
* `Parent: KEY` (Jira Mavisi) -> Subtask'ın direkt bağlı olduğu ana görevin anahtar kodu (Örn: `Parent: MS-1`).
* `Points: X` (Gri) -> Görevin efor karmaşıklığını belirten dinamik etiket.

---

## 💻 Geliştirilen Otomasyon Kodu (import.js)

Jira.csv dosyasındaki çok satırlı açıklamaları, tırnak işaretlerini ve karmaşık sütun yapılarını `csv-parse` kütüphanesiyle ayrıştırıp Jira kodları (MS-1, MS-2) ve subtask-parent ilişkisiyle GitHub API'sine aktaran güncel script aşağıdadır:

```javascript
const fs = require("fs");
const axios = require("axios");
const { parse } = require("csv-parse/sync");

const GITHUB_TOKEN = "YOUR_GITHUB_TOKEN_HERE"; 
const REPO_OWNER = "yemreyazgan";
const REPO_NAME = "jira-migration-test"; 
const CSV_FILE_PATH = "Jira.csv"; 

const colors = {
    "Priority: High": "d93f0b",
    "Priority: Medium": "fbca04",
    "Priority: Low": "0e8a16",
    "Type: Story": "1d76db",
    "Type: Task": "5319e7",
    "Type: Bug": "d73a4a",
    "Type: Subtask": "0e8a16"
};

async function ensureLabel(labelName, customColor) {
    if (!labelName || labelName.endsWith(':')) return;
    try {
        await axios.post(`https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/labels`, {
            name: labelName,
            color: customColor || colors[labelName] || "cccccc"
        }, {
            headers: { "Authorization": `token ${GITHUB_TOKEN}`, "User-Agent": "Migration" }
        });
    } catch (e) {
        // Label zaten varsa pas geç
    }
}

async function startMigration() {
    try {
        console.log("📄 Jira.csv okunuyor ve MS-1 / MS-2 kodları başlıklara işleniyor...");

        let content = fs.readFileSync(CSV_FILE_PATH, "utf8");
        if (content.startsWith('\uFEFF')) {
            content = content.slice(1);
        }

        const records = parse(content, {
            columns: true,
            skip_empty_lines: true,
            trim: true
        });

        const parentTasks = {};
        const rawSubtasks = [];

        records.forEach(row => {
            const issueType = row["Issue Type"] ? row["Issue Type"].trim() : "";
            const issueKey = row["Issue key"] ? row["Issue key"].trim() : "";
            let summary = row["Summary"] ? row["Summary"].trim() : "";

            if (!summary) return;

            summary = summary.replace(/^Task\s*\d+:\s*/i, "");

            if (issueType === "Subtask") {
                const parentKey = row["Parent key"] ? row["Parent key"].trim() : "";
                rawSubtasks.push({
                    key: issueKey,
                    summary: summary,
                    parentKey: parentKey,
                    priority: row["Priority"] || "Medium"
                });
            } else {
                const rawDesc = row["Description"] || "";

                const affectsMatch = rawDesc.match(/\*?Affects Version:\*?\s*([^\n]+)/i);
                const buildMatch = rawDesc.match(/\*?Build Info:\*?\s*([^\n]+)/i);

                const affectsVersion = affectsMatch ? affectsMatch[1].replace(/\*/g, "").trim() : "N/A";
                const buildInfo = buildMatch ? buildMatch[1].replace(/\*/g, "").trim() : "N/A";

                let cleanDesc = rawDesc
                    .replace(/\*?Affects Version:\*?[^\n]+\n*/gi, "")
                    .replace(/\*?Build Info:\*?[^\n]+\n*/gi, "")
                    .replace(/^---/gm, "")
                    .trim();

                const storyPoints = row["Custom field (Story point estimate)"] || "N/A";
                const fixVersion = row["Fix versions"] || "N/A";
                const priority = row["Priority"] || "Medium";
                
                let rawDueDate = row["Due date"] || "N/A";
                let dueDate = (rawDueDate && rawDueDate !== "N/A") ? rawDueDate.split(" ")[0] : "N/A";

                parentTasks[issueKey] = {
                    key: issueKey,
                    type: issueType || "Task",
                    summary: summary,
                    description: cleanDesc,
                    storyPoints: storyPoints !== "" ? storyPoints : "N/A",
                    fixVersion: fixVersion !== "" ? fixVersion : "N/A",
                    affectsVersion: affectsVersion,
                    buildInfo: buildInfo,
                    dueDate: dueDate,
                    priority: priority,
                    subtaskIssues: []
                };
            }
        });

        console.log("🚀 1. Adım: Subtask'lar Parent Jira kodlarıyla oluşturuluyor...");
        await ensureLabel("Type: Subtask");
        await ensureLabel("Jira-Migration");

        for (const st of rawSubtasks) {
            const stPriorityLabel = `Priority: ${st.priority}`;
            const parentLabel = `Parent: ${st.parentKey}`;
            await ensureLabel(stPriorityLabel);
            await ensureLabel(parentLabel, "0052cc");

            const parentSummary = parentTasks[st.parentKey] ? parentTasks[st.parentKey].summary : st.parentKey;

            const stPayload = {
                title: `[${st.key}] [Subtask of ${st.parentKey}] ${st.summary}`,
                body: `### 📌 Alt Görev Detayı\n- **Jira Key:** \`${st.key}\`\n- **Bağlı Olduğu Ana Task:** \`${st.parentKey}\` - ${parentSummary}\n\n*Jira Migration Tool ile Otomatik Oluşturuldu*`,
                assignees: ["yemreyazgan"],
                labels: ["Jira-Migration", "Type: Subtask", parentLabel, stPriorityLabel]
            };

            const stResponse = await axios.post(`https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/issues`, stPayload, {
                headers: { 
                    "Authorization": `token ${GITHUB_TOKEN}`, 
                    "Accept": "application/vnd.github.v3+json", 
                    "User-Agent": "Migration" 
                }
            });

            const createdIssueNumber = stResponse.data.number;

            if (parentTasks[st.parentKey]) {
                parentTasks[st.parentKey].subtaskIssues.push({
                    number: createdIssueNumber,
                    summary: st.summary,
                    key: st.key
                });
            }
        }

        console.log("\n🚀 2. Adım: Ana Görevler Jira Kodlarıyla (MS-1, MS-2...) yükleniyor...\n");

        for (const key in parentTasks) {
            const t = parentTasks[key];

            const priorityLabel = `Priority: ${t.priority}`;
            const typeLabel = `Type: ${t.type}`;
            const pointLabel = `Points: ${t.storyPoints}`;

            await ensureLabel(priorityLabel);
            await ensureLabel(typeLabel);
            if (t.storyPoints !== "N/A") await ensureLabel(pointLabel);

            let subtasksMarkdown = "";
            if (t.subtaskIssues.length > 0) {
                subtasksMarkdown = t.subtaskIssues.map(s => `- [ ] #${s.number} - **[${s.key}]** ${s.summary}`).join("\n");
            } else {
                subtasksMarkdown = "_Alt görev bulunmuyor._";
            }

            const issueBody = `### 📌 Proje Metadataları
| Alan | Değer |
| :--- | :--- |
| **Jira Key** | **${t.key}** |
| **Story Point** | ${t.storyPoints} |
| **Fix Version** | ${t.fixVersion} |
| **Affects Version** | ${t.affectsVersion} |
| **Build Info** | ${t.buildInfo} |
| **Due Date (Bitiş Tarihi)** | **${t.dueDate}** |

---

### 📝 Görev Açıklaması
${t.description}

---

### 🔲 Subtasks (Alt Görevler)
${subtasksMarkdown}

---
*Jira Migration Tool ile Otomatik Oluşturuldu (${t.key})*`;

            const labelsToApply = ["Jira-Migration"];
            if (t.priority) labelsToApply.push(priorityLabel);
            if (t.type) labelsToApply.push(typeLabel);
            if (t.storyPoints !== "N/A") labelsToApply.push(pointLabel);

            const issuePayload = {
                title: `[${t.key}] ${t.summary}`,
                body: issueBody,
                assignees: ["yemreyazgan"],
                labels: labelsToApply
            };

            await axios.post(`https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/issues`, issuePayload, {
                headers: { 
                    "Authorization": `token ${GITHUB_TOKEN}`, 
                    "Accept": "application/vnd.github.v3+json", 
                    "User-Agent": "Migration" 
                }
            });

            console.log(`✅ [${t.key}] ${t.summary}`);
        }

        console.log("\n🎉 [MİGRASYON TAMAMLANDI]");

    } catch (e) {
        console.error("❌ Hata:", e.response ? e.response.data : e.message);
    }
}

startMigration();
```
