# 📊 Jira to GitHub Issue Migration Project

Bu depo, Jira üzerindeki görevlerin, iş tiplerinin, öncelik derecelerinin, alt görevlerin (subtask), efor puanlarının (Story Point), versiyon bilgilerinin ve proje takvimlerinin GitHub Issues ve Milestones mimarisine kurumsal standartlarda taşınması (migration) amacıyla geliştirilmiştir.

---

## 🎯 Proje Yapısı ve Eşleşmeler

Jira'daki mevcut proje yönetim yapısı, GitHub mimarisine birebir ve dinamik olarak şu şekilde entegre edilmiştir:

* **Görev Başlıkları (Title):** Jira'daki her bir task, özet başlıklarıyla birlikte GitHub Issue başlığı olarak aktarılmıştır.
* **Proje Metadataları (Metadata Table):** Her Issue'nun en üstüne kurumsal formatta Markdown tablosu eklenmiştir:
  * **Story Point:** Efor puanı.
  * **Fix Version & Affects Version:** Sürüm ve ortam bilgileri.
  * **Build Info:** Derleme/Build detayları.
  * **Due Date:** Bitiş/Teslim tarihi.
* **İş İçerikleri (Description):** Görevlerin teknik detayları ve açıklamaları hatasız bir şekilde biçimlendirilerek taşınmıştır.
* **Alt Görevler (Subtasks):** Jira'da ana göreve bağlı açılan subtask'lar, GitHub Issue içerisinde etkileşimli/tıklanabilir **Checklist (`- [ ]`)** formatına dönüştürülmüştür.
* **Proje Takvimi (Milestone):** GitHub üzerinde **"Sprint 1: Temmuz Teslimatları"** adında süreli bir Kilometre Taşı (Milestone) oluşturulmuş ve tüm görevler bu takvime bağlanmıştır.
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

### 3. Efor Puanı (Story Point)
* `Points: X` (Gri) -> Görevin efor karmaşıklığını belirten dinamik etiket.

---

## 💻 Geliştirilen Otomasyon Kodu (import.js)

Jira.csv dosyasındaki çok satırlı açıklamaları, tırnak işaretlerini ve karmaşık sütun yapılarını profesyonel `csv-parse` kütüphanesiyle ayrıştırıp GitHub API'sine aktaran güncel script aşağıdadır:

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
    "Type: Bug": "d73a4a"
};

async function ensureLabel(labelName) {
    if (!labelName || labelName.endsWith(':')) return;
    try {
        await axios.post(`https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/labels`, {
            name: labelName,
            color: colors[labelName] || "cccccc"
        }, {
            headers: { "Authorization": `token ${GITHUB_TOKEN}`, "User-Agent": "Migration" }
        });
    } catch (e) {
        // Etiket varsa pas geç
    }
}

async function startMigration() {
    try {
        console.log("📄 Jira.csv profesyonel parser ile okunuyor...");

        let content = fs.readFileSync(CSV_FILE_PATH, "utf8");
        if (content.startsWith('\uFEFF')) {
            content = content.slice(1);
        }

        const records = parse(content, {
            columns: true,
            skip_empty_lines: true,
            trim: true
        });

        const tasks = {};
        const subtasks = [];

        records.forEach(row => {
            const issueType = row["Issue Type"] ? row["Issue Type"].trim() : "";
            const issueKey = row["Issue key"] ? row["Issue key"].trim() : "";
            const summary = row["Summary"] ? row["Summary"].trim() : "";

            if (!summary) return;

            if (issueType === "Subtask") {
                const parentKey = row["Parent key"] ? row["Parent key"].trim() : "";
                subtasks.push({ summary, parentKey });
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

                tasks[issueKey] = {
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
                    subtasks: []
                };
            }
        });

        subtasks.forEach(st => {
            if (tasks[st.parentKey]) {
                tasks[st.parentKey].subtasks.push(st.summary);
            }
        });

        console.log("🚀 1. Adım: Sprint Milestone kontrol ediliyor...");
        let milestoneNumber;
        try {
            const milestoneResponse = await axios.post(`https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/milestones`, {
                title: "Sprint 1: Temmuz Teslimatları",
                description: "Jira'dan aktarılan görevlerin teslimat takvimi.",
                due_on: "2026-07-25T18:00:00Z"
            }, {
                headers: { 
                    "Authorization": `token ${GITHUB_TOKEN}`, 
                    "Accept": "application/vnd.github.v3+json", 
                    "User-Agent": "Migration" 
                }
            });
            milestoneNumber = milestoneResponse.data.number;
        } catch (e) {
            console.log("ℹ️ Milestone zaten mevcut.");
        }

        console.log("🚀 2. Adım: Görevler GitHub'a aktarılıyor...\n");

        for (const key in tasks) {
            const t = tasks[key];

            const priorityLabel = `Priority: ${t.priority}`;
            const typeLabel = `Type: ${t.type}`;
            const pointLabel = `Points: ${t.storyPoints}`;

            await ensureLabel(priorityLabel);
            await ensureLabel(typeLabel);
            if (t.storyPoints !== "N/A") await ensureLabel(pointLabel);
            await ensureLabel("Jira-Migration");

            let subtasksMarkdown = "";
            if (t.subtasks.length > 0) {
                subtasksMarkdown = t.subtasks.map(s => `- [ ] ${s}`).join("\n");
            } else {
                subtasksMarkdown = "_Alt görev bulunmuyor._";
            }

            const issueBody = `### 📌 Proje Metadataları
| Alan | Değer |
| :--- | :--- |
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
                title: t.summary,
                body: issueBody,
                assignees: ["yemreyazgan"],
                labels: labelsToApply
            };

            if (milestoneNumber) {
                issuePayload.milestone = milestoneNumber;
            }

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
