# job_seeker_ro_spider — Scrapers Index

Toate scraper-ele Node.js derivate din template-ul EPAM pentru peviitor.ro.

**Template sursă:** [epam-systems-international-srl-nodejs-scraper](https://github.com/sebiboga/epam-systems-international-srl-nodejs-scraper) (CIF: 33159615, brand: EPAM)  
**AI Factory:** [AI-Factory-job-seeker-ro-spider](https://github.com/sebiboga/AI-Factory-job-seeker-ro-spider)  
**Ultima actualizare:** 2026-06-21

---

## Derivate active (template → badge "Generated from")

| # | Brand | Companie | CIF | Metodă scraping | Repository | Pages |
|---|---|---|---|---|---|---|
| 1 | **Utilben** | UTILBEN SRL | 18643343 | API JSON | [↗](https://github.com/sebiboga/utilben-srl-nodejs-scraper) | [↗](https://sebiboga.github.io/utilben-srl-nodejs-scraper/) |
| 2 | **AD/01** | Ahold Delhaize Technologies SRL | 49544242 | HTML/cheerio | [↗](https://github.com/sebiboga/ahold-delhaize-technologies-srl-nodejs-scraper) | [↗](https://sebiboga.github.io/ahold-delhaize-technologies-srl-nodejs-scraper/) |
| 3 | **ARTSOFT CONSULT** | ArtSoft Consult SRL | 15997630 | HTML/cheerio | [↗](https://github.com/sebiboga/artsoft-consult-srl-nodejs-scraper) | [↗](https://sebiboga.github.io/artsoft-consult-srl-nodejs-scraper/) |
| 3 | **COERA** | COERA BC SRL | 32519996 | HTML/cheerio | [↗](https://github.com/sebiboga/coera-bc-srl-nodejs-scraper) | [↗](https://sebiboga.github.io/coera-bc-srl-nodejs-scraper/) |
| 4 | **Continental Hotels** | CONTINENTAL HOTELS SA | 1559737 | POST AJAX → HTML | [↗](https://github.com/sebiboga/continental-hotels-srl-nodejs-scraper) | [↗](https://sebiboga.github.io/continental-hotels-srl-nodejs-scraper/) |
| 5 | **EMERSON** | EMERSON SRL | 18284762 | Oracle Cloud HCM API | [↗](https://github.com/sebiboga/emerson-srl-nodejs-scraper) | [↗](https://sebiboga.github.io/emerson-srl-nodejs-scraper/) |
| 6 | **GAMINVEST** | GAMINVEST SRL | 21913994 | HTML/cheerio | [↗](https://github.com/sebiboga/gaminvest-srl-nodejs-scraper) | [↗](https://sebiboga.github.io/gaminvest-srl-nodejs-scraper/) |
| 7 | **Mejix** | MEJIX SRL | 17372688 | HTML/cheerio | [↗](https://github.com/sebiboga/mejix-srl-nodejs-scraper) | [↗](https://sebiboga.github.io/mejix-srl-nodejs-scraper/) |
| 8 | **RAPEL** | RAPEL SRL | 5665609 | jobRapid.ro HTML | [↗](https://github.com/sebiboga/rapel-srl-nodejs-scraper) | [↗](https://sebiboga.github.io/rapel-srl-nodejs-scraper/) |
| 9 | **ROPARDO** | ROPARDO SRL | 5415866 | HTML/cheerio | [↗](https://github.com/sebiboga/ropardo-srl-nodejs-scraper) | [↗](https://sebiboga.github.io/ropardo-srl-nodejs-scraper/) |
| 10 | **sennder** | SENNDER BUCHAREST S.R.L. | 45780151 | Gem ATS API | [↗](https://github.com/sebiboga/sennder-bucharest-srl-nodejs-scraper) | [↗](https://sebiboga.github.io/sennder-bucharest-srl-nodejs-scraper/) |
| 11 | **Talent Matchmakers** | TALENT MATCHMAKERS S.R.L. | 38460545 | Teamtailor HTML | [↗](https://github.com/sebiboga/talent-matchmakers-srl-nodejs-scraper) | [↗](https://sebiboga.github.io/talent-matchmakers-srl-nodejs-scraper/) |
| 12 | **TEC Agency** | TEC SOFTWARE SOLUTIONS SRL | 32971419 | HTML/cheerio | [↗](https://github.com/sebiboga/tec-software-solutions-srl-nodejs-scraper) | [↗](https://sebiboga.github.io/tec-software-solutions-srl-nodejs-scraper/) |
| 13 | **FARMEC** | FARMEC SA | 199150 | HTML/cheerio + eJobs | [↗](https://github.com/sebiboga/farmec-sa-nodejs-scraper) | [↗](https://sebiboga.github.io/farmec-sa-nodejs-scraper/) |
| 15 | **CONNATIX** | CONNATIX NATIVE EXCHANGE ROMANIA SRL | 35861771 | Greenhouse API (JSON fetch) | [↗](https://github.com/sebiboga/connatix-native-exchange-romania-srl-nodejs-scraper) | [↗](https://sebiboga.github.io/connatix-native-exchange-romania-srl-nodejs-scraper/) |

**Total derivate active: 15** (14 create din template cu badge "Generated from EPAM" + 1 pre-exista configurat manual)

---

## Despre

Acest index este generat și menținut de [AI Factory](https://github.com/sebiboga/AI-Factory-job-seeker-ro-spider). Datele sunt în [`scrapers.json`](scrapers.json) în același director. Fiecare scraper din listă:

- Este creat din template-ul EPAM via GitHub `--template` (primește badge-ul "Generated from")
- Rulează zilnic prin GitHub Actions și publică job-uri în **peviitor.ro** via API SOLR
- Are GitHub Pages activat pe branch-ul `main`, folder `/docs`
- Are `SOLR_AUTH` configurat ca secret în repo

Datele complete în format JSON: [`scrapers.json`](scrapers.json)
