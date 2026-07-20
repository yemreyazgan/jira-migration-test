# 📊 Jira to GitHub Issue Migration Project

Bu depo, Jira üzerindeki görevlerin, iş tiplerinin, öncelik derecelerinin ve proje takvimlerinin GitHub Issues ve Milestones mimarisine kurumsal standartlarda taşınması (migration) amacıyla hazırlanmıştır.

---

## 🎯 Proje Yapısı ve Eşleşmeler

Jira'daki mevcut proje yönetim yapısı, GitHub mimarisine birebir ve dinamik olarak şu şekilde entegre edilmiştir:

* **Görev Başlıkları (Title):** Jira'daki her bir task, özet başlıklarıyla birlikte GitHub Issue başlığı olarak aktarılmıştır.
* **İş İçerikleri (Description):** Görevlerin detayları, açıklamaları ve teknik metinleri hatasız bir şekilde Issue gövdesine (Body) taşınmıştır.
* **Proje Takvimi (Milestone):** Jira'daki teslim tarihleri baz alınarak GitHub üzerinde **"Sprint 1: Temmuz Teslimatları"** adında süreli bir Kilometre Taşı (Milestone) oluşturulmuş ve tüm görevler bu takvime bağlanmıştır.
* **Sorumlu Atamaları (Assignee):** Taşınan tüm görevler, projenin takibi ve yönetimi için ilgili geliştiriciye (`yemreyazgan`) otomatik olarak atanmıştır.

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

---

## 💻 Geliştirilen Otomasyon Kodu (import.js)

Jira.csv dosyasındaki verileri ayıklayıp GitHub API'si üzerinden repoya işleyen ana kaynak kod aşağıdadır:

```javascript
const fs = require("fs");
const axios = require("axios");

const GITHUB_TOKEN = "YOUR_GITHUB_TOKEN_HERE"; 
const REPO_OWNER = "yemreyazgan";
const REPO_NAME = "jira-migration-test"; 
const CSV_FILE_PATH = "Jira.csv"; 

// 1. Dosyayı oku ve BOM temizliği yap
let content = fs.readFileSync(CSV_FILE_PATH, "utf8");
if (content.startsWith('\uFEFF')) {
    content = content.slice(1);
}

// 2. Blokları ayır
const task1Index = content.indexOf("Task 1:");
const task2Index = content.indexOf("Task 2:");
const task3Index = content.indexOf("Task 3:");

if (task1Index === -1 || task2Index === -1 || task3Index === -1) {
    console.error("Hata: Tasklar bulunamadı!");
    process.exit(1);
}

const task1Block = content.substring(task1Index, task2Index);
const task2Block = content.substring(task2Index, task3Index);
const task3Block = content.substring(task3Index);

// GitHub etiket renkleri (Hex kodu)
const colors = {
    "Priority: High": "d93f0b",   // Kırmızı
    "Priority: Medium": "fbca04", // Sarı
    "Priority: Low": "0e8a16",    // Yeşil
    "Type: Story": "1d76db",      // Mavi
    "Type: Task": "5319e7"        // Mor
};

// GitHub'da etiket yoksa otomatik oluşturan yardımcı fonksiyon
async function ensureLabel(labelName) {
    try {
        await axios.post(`[https://api.github.com/repos/$](https://api.github.com/repos/$){REPO_OWNER}/${REPO_NAME}/labels`, {
            name: labelName,
            color: colors[labelName] || "ffffff"
        }, {
            headers: { "Authorization": `token ${GITHUB_TOKEN}`, "User-Agent": "Migration" }
        });
    } catch (e) {
        // Etiket zaten varsa hata verir, görmezden gelebiliriz
    }
}

function parseBlock(block) {
    const lines = block.split(/\r?\n/);
    
    // Satırın başındaki verilerden başlığı temizleyerek alıyoruz
    let firstLine = lines[0];
    let title = firstLine.split(",")[0].replace(/"/g, "").trim();
    if(title.startsWith("MS-")) {
        title = title.substring(title.indexOf("Task"));
    }

    // Blok içindeki virgülle ayrılmış alanlardan Issue Type ve Priority'yi dinamik yakala
    const parts = firstLine.split(",");
    const issueType = parts[3] ? parts[3].replace(/"/g, "").trim() : "Task";
    
    let priority = "Medium";
    if (firstLine.includes("High")) priority = "High";
    if (firstLine.includes("Low")) priority = "Low";
    if (firstLine.includes("Medium")) priority = "Medium";

    // Açıklama metnini ayıkla
    let description = block.substring(block.indexOf("Task")).trim();
    const firstLineEnd = description.indexOf("\n");
    if (firstLineEnd !== -1) {
        description = description.substring(firstLineEnd).trim();
    }

    const cleanEnd = description.lastIndexOf(",0|i000");
    if (cleanEnd !== -1) description = description.substring(0, cleanEnd);

    description = description.replace(/^"/, "").replace(/"$/, "").trim();
    description = description.replace(/""/g, '"');

    return { 
        title, 
        description, 
        priorityLabel: `Priority: ${priority}`, 
        typeLabel: `Type: ${issueType}` 
    };
}

const finalTasks = [
    parseBlock(task1Block),
    parseBlock(task2Block),
    parseBlock(task3Block)
];

async function startMigration() {
    try {
        console.log("🚀 1. Adım: GitHub üzerinde Sprint Milestone oluşturuluyor...");
        const milestoneResponse = await axios.post(`[https://api.github.com/repos/$](https://api.github.com/repos/$){REPO_OWNER}/${REPO_NAME}/milestones`, {
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

        const milestoneNumber = milestoneResponse.data.number;
        console.log(`✅ Milestone başarıyla bağlandı! ID: ${milestoneNumber}\n`);

        console.log("🚀 2. Adım: Gerekli kurumsal etiketler GitHub'a tanımlanıyor...");
        for (const task of finalTasks) {
            await ensureLabel(task.priorityLabel);
            await ensureLabel(task.typeLabel);
        }
        await ensureLabel("Jira-Migration");

        console.log("\n🚀 3. Adım: Görevler otomatik atama (Assignee) ve etiketlerle yükleniyor...");
        for (const task of finalTasks) {
            await axios.post(`[https://api.github.com/repos/$](https://api.github.com/repos/$){REPO_OWNER}/${REPO_NAME}/issues`, {
                title: task.title,          
                body: task.description,     
                milestone: milestoneNumber,
                assignees: ["yemreyazgan"], 
                labels: ["Jira-Migration", task.priorityLabel, task.typeLabel] 
            }, {
                headers: { 
                    "Authorization": `token ${GITHUB_TOKEN}`, 
                    "Accept": "application/vnd.github.v3+json", 
                    "User-Agent": "Migration" 
                }
            });
            console.log(`✅ Tam donanımlı olarak aktarıldı: ${task.title}`);
        }
        console.log("\n[MİGRASYON TAMAMLANDI]");

    } catch (e) {
        console.error("❌ Kritik Hata:", e.response ? e.response.data : e.message);
    }
}

startMigration();