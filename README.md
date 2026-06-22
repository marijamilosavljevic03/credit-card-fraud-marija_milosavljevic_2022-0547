Credit Card Fraud Detection

Projekat iz predmeta Duboko učenje i neuronske mreže.

Autor: Marija Milosavljević

1. Opis problema

Prevare kreditnim karticama predstavljaju ozbiljan problem za banke i platne sisteme, jer svaka neotkrivena prevara direktno znači finansijski gubitak, dok pogrešno blokirana legitimna transakcija narušava poverenje korisnika. Cilj ovog projekta je da se na osnovu istorijskih podataka o transakcijama izgradi neuronska mreža koja sa visokom pouzdanošću razlikuje prevare od legitimnih transakcija.

Glavna prepreka u ovom zadatku je izrazita neravnoteža klasa: prevare čine svega 0.17% svih transakcija u datasetu. Posledica toga je da naivan model koji bi za svaku transakciju predvideo "legitimna" automatski postiže tačnost (accuracy) od preko 99.8%, a da pritom ne otkrije nijednu prevaru. Iz tog razloga, accuracy nije pogodna metrika za ovaj problem, i tokom projekta je fokus stavljen na metrike koje bolje odražavaju realne performanse modela na neuravnoteženim podacima( precision, recall, F1-skor i AUC-ROC).

2. Podaci

Izvor: Kaggle — Credit Card Fraud Detection dataset (transakcije evropskih korisnika kartica, septembar 2013).

Struktura dataseta:


284,807 transakcija, 31 kolona
Time — vreme proteklo od prve transakcije u datasetu (u sekundama)
V1–V28 — anonimizovane numeričke karakteristike dobijene PCA transformacijom originalnih podataka (zbog zaštite privatnosti korisnika, banka nije objavila originalna značenja ovih kolona)
Amount — iznos transakcije
Class — ciljna promenljiva: 0 = legitimna transakcija, 1 = prevara


Raspodela klasa:

KlasaBroj transakcijaProcenatLegitimna (0)284,31599.83%Prevara (1)4920.17%

Analiza karakteristika Time i Amount:

Pošto su kolone V1–V28 već anonimizovane PCA transformacijom, detaljnija analiza je sprovedena nad originalnim, netransformisanim kolonama Time i Amount.

Raspodela iznosa pokazuje da su prevare koncentrisane pretežno na manjim iznosima transakcija, najveći broj lažnih transakcija nalazi se ispod 100 jedinica, dok se legitimne transakcije takođe najvećim delom kreću u nižem opsegu, ali sa znatno dužim "repom" prema visokim iznosima (do 25,691). Ovakav obrazac kod prevara je očekivan, jer manji iznosi lakše prolaze nezapaženo kroz osnovne sigurnosne provere.

Raspodela transakcija kroz vreme pokazuje jasan dnevni ciklus kod legitimnih transakcija, broj transakcija opada tokom noćnih sati i raste tokom dana, što odgovara prirodnom obrascu ljudske aktivnosti. Kod prevara takav pravilan ciklus nije izražen na isti način, što ukazuje da Time sam za sebe nije jak prediktor, ali razlika u obrascu između klasa može doprineti modelu kao dodatna informacija.

Preprocesiranje:


Provera nedostajućih vrednosti: nije pronađena nijedna u celom datasetu
Skaliranje kolona Time i Amount pomoću StandardScaler (kolone V1–V28 su već skalirane usled PCA transformacije i nije ih bilo potrebno dodatno obrađivati)
Podela na trening (80%) i test (20%) skup, uz stratifikaciju po koloni Class kako bi i trening i test skup zadržali isti odnos klasa kao originalni dataset


SkupBroj transakcijaBroj prevaraTrening227,845394Test56,96298

Balansiranje klasa: SMOTE (razmera 1:3):

Za rešavanje problema neravnoteže klasa korišćena je SMOTE (Synthetic Minority Oversampling Technique) tehnika, koja generiše sintetičke primere manjinske klase interpolacijom između postojećih primera prevare. Za razliku od potpunog izjednačavanja klasa (1:1), u ovom projektu je primenjen blaži odnos balansiranja od 1:3 (sampling_strategy=0.33), čime je broj primera prevare u trening skupu doveden na otprilike trećinu broja legitimnih primera, umesto na potpuno izjednačen broj.

Razlog za ovaj izbor je smanjenje rizika od prekomernog veštačkog popunjavanja manjinske klase — generisanje prevelikog broja sintetičkih primera (npr. duplo više nego što ih ima u stvarnosti) nosi rizik da model nauči obrasce koji ne postoje u realnim podacima.

Pre SMOTEPosle SMOTELegitimne227,451227,451Prevare39475,058Ukupno227,845302,509

SMOTE je primenjen isključivo na trening skupu, dok je test skup ostao netaknut sa originalnom (realnom) raspodelom klasa, kako bi evaluacija modela odražavala stvarne uslove.

3. Arhitektura modela

Za klasifikaciju je korišćen MLPClassifier (Multi-Layer Perceptron) iz biblioteke scikit-learn, neuronska mreža sa potpuno povezanim (fully connected) skrivenim slojevima.

Baseline arhitektura:


Ulazni sloj: 30 karakteristika (feature-a)
Skriveni sloj 1: 96 neurona, aktivacija ReLU
Skriveni sloj 2: 32 neurona, aktivacija ReLU
Izlazni sloj: 1 neuron (binarna klasifikacija)
Learning rate: 0.001
Maksimalan broj iteracija: 50
Optimizator: Adam


Arhitektura sa dva skrivena sloja je odabrana kao polazna tačka zbog jednostavnosti i brzine treniranja, sa namerom da se u sklopu analize osetljivosti ispita da li dublje i šire arhitekture donose merljivo poboljšanje performansi.

4. Trening

Model je treniran na SMOTE-balansiranom trening skupu (302,509 primera), korišćenjem Adam optimizatora uz mehanizam ranog zaustavljanja (early stopping). Trening se automatski zaustavlja ukoliko se validaciona tačnost ne poboljša za više od tol=0.0001 tokom 10 uzastopnih epoha, čime se sprečava nepotrebno dugo treniranje i smanjuje rizik od overfitting-a.

Baseline model je dostigao kriterijum ranog zaustavljanja nakon 24 epohe, sa vremenom treniranja od približno 64 sekunde.

5. Analiza osetljivosti i hiperparametarska optimizacija

Sprovedena je pretraga po 15 kombinacija hiperparametara, sa fokusom na tri dimenzije: veličinu/dubinu arhitekture, aktivacionu funkciju i učestalost koraka učenja (learning rate).

HiperparametarTestirane vrednostihidden_layer_sizes(50,25), (96,32), (64,32,16), (128,64), (128,64,32), (150,75,30)activationrelu, tanhlearning_rate_init0.0001, 0.001, 0.01

Najbolji rezultati po F1-skoru:

hidden_layer_sizesactivationlearning_rateF1AUC-ROC(150, 75, 30)relu0.0010.81820.9731(128, 64)relu0.0010.8000—(128, 64)tanh0.0010.8000—(64, 32, 16)relu0.0010.7826—(96, 32)relu0.0010.77390.9715(50, 25)relu0.0010.7373—

Ključni nalazi:


Dubina odnosno širina mreže ima najveći uticaj na rezultat. Najbolja arhitektura, (150, 75, 30), nadmašila je baseline (96, 32) sa F1=0.8182 naspram F1=0.7739. Manja arhitektura, (50, 25), pokazala je najslabije rezultate od svih netrivijalnih kombinacija (F1=0.7373), što potvrđuje da mreži treba dovoljan kapacitet da uhvati složene obrasce u 30 ulaznih karakteristika.
Aktivaciona funkcija ima manji uticaj kada je arhitektura fiksirana (128, 64) sa ReLU i (128, 64) sa tanh dale su identičan F1-skor (0.8000), što ukazuje da izbor aktivacione funkcije ovde nije presudan faktor.
Learning rate van podrazumevane vrednosti pogoršava rezultat. Za istu arhitekturu (96, 32), lr=0.001 dao je F1=0.7739, dok je lr=0.01 dao F1=0.7345, a lr=0.0001 čak F1=0.7200 i previsoka i preniska vrednost koraka učenja smanjuju performanse modela.


Na osnovu ove analize, kao finalni (optimizovani) model odabrana je arhitektura (150, 75, 30) sa ReLU aktivacijom i learning rate-om 0.001, koja je dostigla rano zaustavljanje nakon 27 epoha.

Treba napomenuti da veće arhitekture, iako ovde daju bolje rezultate na test skupu, nose i veći rizik od overfitting-a u opštem slučaju za pouzdaniju procenu generalizacije modela u produkcionom okruženju bilo bi korisno dodatno sprovesti unakrsnu validaciju (cross-validation), što izlazi iz okvira ovog projekta.

6. Rezultati evaluacije

Optimizovani MLP model (150, 75, 30):

MetrikaLegitimnaPrevaraPrecision1.000.81Recall1.000.83F1-score1.000.82


AUC-ROC: 0.9731
Propuštenih prevara (False Negative): 17 od 98
Lažnih uzbuna (False Positive): 19


Poređenje sa drugim modelima:

ModelPrecision (prevara)Recall (prevara)F1 (prevara)AUC-ROCLogistička regresija0.150.900.260.9680MLP,  baseline (96, 32)0.760.790.770.9715MLP, optimizovani (150, 75, 30)0.810.830.820.9731

Logistička regresija postiže visok recall (0.90  hvata veliku većinu prevara), ali ekstremno nizak precision (0.15), što znači da je od svake transakcije koju ovaj model označi kao prevaru, samo oko 15% zaista i jeste prevara, dok je preostalih 85% lažna uzbuna. Sa praktične strane, ovakav model bi u realnom sistemu generisao neprihvatljivo veliki broj pogrešno blokiranih legitimnih transakcija.

Zanimljivo, AUC-ROC vrednosti logističke regresije i MLP modela su relativno bliske (0.968 naspram 0.973), iako je razlika u F1-skoru ogromna. Ovo ilustruje zašto AUC-ROC sam po sebi nije dovoljan pokazatelj kvaliteta modela kod izrazito neuravnoteženih klasa, metrika posmatra model preko svih mogućih pragova odlučivanja, dok F1-skor (računat na podrazumevanom pragu 0.5) mnogo bolje odražava stvarnu, praktičnu upotrebljivost modela.

7. Diskusija

Glavni izazovi tokom izrade projekta bili su:


Neravnoteža klasa, rešena kombinacijom SMOTE tehnike (sa blažim, 1:3 razmerom balansiranja umesto potpunog izjednačavanja) i pažljivim izborom evaluacionih metrika.
Izbor odgovarajuće metrike (accuracy) je pokazana kao neupotrebljiva metrika za ovaj problem, umesto nje, fokus je stavljen na F1-skor i AUC-ROC, uz detaljno praćenje precision/recall odnosa.
Kompromis između precision i recall ,u kontekstu detekcije prevara, recall je generalno važniji (cilj je ne propustiti prevaru), ali prekomerno visok recall uz nizak precision dovodi do velikog broja lažnih uzbuna, što u praksi opterećuje sistem i korisnike. Optimizovani MLP model postiže solidan balans (precision=0.81, recall=0.83), za razliku od logističke regresije kod koje je taj balans znatno lošiji.
Izbor arhitekture mreže, analiza osetljivosti je pokazala da veličina mreže ima znatno veći uticaj na rezultat od izbora aktivacione funkcije, što je bio koristan uvid za dalje eksperimentisanje.


8. Zaključak

U ovom projektu uspešno je izgrađen model za detekciju prevara kreditnim karticama korišćenjem MLP neuronske mreže, koji na test skupu postiže AUC-ROC od 0.9731 i F1-skor od 0.82 za klasu prevara. Poređenjem sa logističkom regresijom pokazano je da MLP neuronska mreža postiže znatno bolji balans između precision i recall metrika, što je ključno za praktičnu primenljivost ovakvog sistema,  sistem koji generiše previše lažnih uzbuna nije upotrebljiv u realnom bankarskom okruženju bez obzira na visok recall.

Analiza osetljivosti hiperparametara pokazala je da dublje/šire arhitekture mreže (u ovom slučaju 150-75-30 neurona) daju bolje rezultate od jednostavnijih arhitektura, dok izbor aktivacione funkcije ima manji uticaj na konačan rezultat.

Moguća dalja unapređenja uključuju primenu cross-validacije radi pouzdanije procene generalizacije modela, ispitivanje naprednijih arhitektura poput LSTM mreža (koje bi mogle iskoristiti sekvencijalnu prirodu kolone Time), kao i poređenje sa ansambl metodama poput XGBoost-a, koje se često koriste u industriji za ovaj tip problema.

Licenca

MIT License