# Claude-Code-Management

Kurulum: 
curl -fsSL https://claude.ai/install.sh | bash veya Uygulama olarak indirilebilir. 

İstenilen bir dizinde çalıştırılıp, kullanılabilir.

Genel Amaç:
Claude Code yaptığımız işleri hızlandıracak, yazdığımız kodun ortak bir birliktelikte yazılabilmesini sağlyacaktır. 


Şirket içinde örnek kullanım senaryosu; 

Öncelikle Panates içinde bir tane company-claude-management adına bir repo oluşturulur. 

Bu repo'nun file structer; 

├── .claude/
│   ├── agents/
│   │   └── security-reviewer.md
│   ├── skills/
│   │   ├── frontend-isg-peacox/
│   │   │   └── SKILL.md
│   │   ├── frontend-rx-peacox/
│   │   │   └── SKILL.md
│   │   ├── monorepo-navigation/
│   │   │   └── SKILL.md
│   │   ├── opra-api/
│   │   │   └── SKILL.md
│   │   └── react-typescript/
│   │       └── SKILL.md
│   └── CLAUDE.md
└── README.md



CLAUDE.md ne işe yarar ? 

Bir çalışan, rx-tenant-ui projesine girdiğinde (Bu repo aynı zamanda sub module ile şirketin *.md dosyalarının olduğu yetenekleri ve çeşitli şirketi içinde özelleştirilmiş dosyaları çekecektir.) yazacağı kodları rastgele bir ai kullanarak şirket içinde kod yazma tarzından aykırı kod yazmak yerine bu *.md dosyaları sayesinde ai'ın şirkete özelleştirilmiş şekilde tek bir merkezi kod yazma aracı olmuş olacaktır.

SKILL.md ne işe yarar ?

Bir çalışan, rx-tenant için api geliştirirken opra yetkinliklerine sahip değil ise bu skill.md dosyaları sayesinde aslında şirkette daha önceden claude a öğretilen opra'yı çalışan claude sayesinde daha efektif ve hızlı bir şekilde öğrenir.

Agents ne işe yarar ? 

Bir çalışan, küçük işleri olan örneğin "security-review.md","refactor.md" gibi bu işleri birden fazla agentın çalışması ile çok daha efektif ve hızlı şekilde çalışmasıdır.

Claude Review ne işe yarar ? 

Bir şirkette, çalışanların açmış olduğu PR ların hepsinin claude ile kontrol edilmesidir. (Bu PR ları kontrol edip bir comment bırakır PR ları kontrol eden kişi ise bu commentlere göre ister PR onaylar isterse reddeder.)

Not: Bu özellik sadece Teams için geçerlidir. Fakat teams de claude kullanma hakkına dahil değildir, teams'i satın alan kişinin claude admin panelinden bir tane API key generate edip, bu key'i bulunduğu organizasyonuna tanımlamalıdır. Buna PR atılınca kontrol etmesi için ise gerekli repolarda read-only izni verilmelidir. 



Claude Teams ile Claude Pro Ücretlendirme

https://claude.com/pricing