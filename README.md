# BUDAPESTI KOMPLEX SZAKKÉPZÉSI CENTRUM WEISS MANFRÉD SZAKGIMNÁZIUMA, SZAKKÖZÉPISKOLÁJA ÉS KOLLÉGIUMA

## NOTAM PROCESSOR

```
Budapest
2021
```
## Témavezető Készítette

## Kovács László Cserményi Botond


## Tartalom


- Bevezetés
- Fejlesztői dokumentáció
   - Fejlesztői környezet
   - Program felépítése
      - Appok
      - MVT
   - URL mapping
      - Példa az URL mappingra
   - Adatbázis
      - Adatbázis táblák létrehozása
      - Runway tábla
      - Airport tábla
      - Notam tábla
      - User tábla
   - Autentikáció
      - Felhasználó regisztráció
   - Notam model CRUD műveletei
      - Create Notam
      - Retrieve Notam
      - Update Notam
      - Delete Notam
   - NOTAM feldolgozás
      - Célok meghatározása
      - Regexek
      - Segéd függvények
      - Airport tábla feltöltése
      - API hívása
      - Jelentés generálása
   - Tesztdokumentáció
      - Kompatibilitás
- Felhasználói dokumentáció
   - Oldal elérése
   - Fiók kezelés
      - Regisztráció
      - Bejelentkezés
      - Kijelentkezés
   - Bejelentkezéshez kötött oldalak használata
      - NOTAMok listázása
      - Jelentés generálása
   - Korlátozott elérésű oldalak használata
      - Engedélyek
      - Admin felület
      - NOTAMok letöltése, feldolgozása
      - NOTAM hozzáadása
      - NOTAM módosítása
      - NOTAM törlése
      - Lejárt érvényességű NOTAMok eltávolítása
- Összefoglalás
- Irodalomjegyzék
- Köszönetnyilvánítás


## Bevezetés

Szakdolgozatom témájának megválasztásakor egy mindennapos munkahelyi feladat
megkönnyítése, lerövidítése volt a legfőbb szempont. Hozzájárult továbbá az is, hogy a nyílt
közeg számára nincs elérhető megvalósítás ezen feladat bármilyen szintű automatizálására.

Mielőtt rátérnék arra, hogy mi is a program feladata, röviden bemutatom a NOTAMokat.
Ezek a repülésben használt közlemények, teljes nevükön notice to airmen, olyan ideiglenes
vagy rövid határidővel kiadott információkat tartalmaznak, amik a repülés biztonságos
üzemeléséhez szükségesek a személyzet számára. A pilóták által használt repülési terv
jelentős részét a repülőterekre és légterekre kiadott NOTAMok teszik ki.

A program feladata az ICAO (Nemzetközi Polgári Repülési Szervezet) API
szolgáltatásáról lehívni a NOTAMokat, és adatbázisba menteni az érvényességi idő és
kategória szempontjából számunkra érdekes kiadványokat. Feladata továbbá regexek
alkalmazásával kiszűrni belőle az információ lényegét a NOTAM feldolgozása során, az
adatbázis egyik mezője ezt az összefoglalót kell tartalmaznia. Végül pedig jelentést készít az
adatbázisba mentett adatokból, amelyet a honlapon megjelenít, ill. tartalmát emailben elküldi
az előre megadott email címekre.


## Fejlesztői dokumentáció

### Fejlesztői környezet

A program elkészítéséhez a Django keretrendszert választottam két okból. Az egyik, hogy
a Python nyelvet ismerem a leginkább, így a fejlesztés idejét nem növelte meg egy újabb
programnyelv elsajátítása. A másik ok szintén a sebességhez kötődik; a Django sokféle
beépített funkciója nagyban megkönnyíti a fejlesztő munkáját.

Adatbázis-kezelő rendszernek PostgreSQL-t választottam, mivel jól illeszthető mind a
keretrendszerhez, mind pedig a program platformjául választott Herokuval. Webszerverként a
Gunicorn szerepel még a dependenciák listáján, a statikus fájlok szolgáltatására pedig a
WhiteNoise lett felhasználva.

### Program felépítése

#### Appok

```
Minden Django projekt egy vagy több appból áll. Jelen esetben 3 app működik:
• auth – Felhasználók kezelése, regisztráció, beléptetés
• processing – A „fő” app, amelyik a NOTAMok begyűjtését, feldolgozását végzi
• reporting – Az adatbázisba mentett adatokból készít jelentést
```
#### MVT

Az appok az úgynevezett MVT minta mentén vannak felépítve.
Model feladata az adatbázis kezelése. Létrehozza a táblákat, valamint beépített
függvényekkel kezeli a CRUD folyamatokat. View tartalmazza jellemzően a program
logikáját, összeköti a Modelleket a Templatekkel. Utóbbiak a Django Template Language
által dinamikussá tett html fájlok, amelyeket a hozzátartozó View renderel a beérkező kérések
után. Mindezt pedig az URL mapping teszi lehetővé, amelyek relatív url-ekhez kötik a
létrehozott View függvényeket.

View létrehozható függvény vagy osztály alkalmazásával. A fejlesztők feladatának
megkönnyítésére rengeteg beépített, örökölhető osztály áll rendelkezésre.


### URL mapping

#### Példa az URL mappingra

```
processing/urls.py
```
urlpatterns = [
path('', home, name='home'),
path('call/', call, name='call'),
path('upload/', DataView.as_view(), name='upload'),
path('upload-airports/', AirportView.as_view(),
name='upload_airports'),
path('select/', NotamSelectView.as_view(), name='notam_select'),
path('notams/', NotamListView.as_view(), name='notam_list'),
path('create-notam/', NotamCreateView.as_view(),
name='notam_create'),
path('<int:pk>/update/', NotamUpdateView.as_view(),
name='notam_update'),
path('<int:pk>/delete/', NotamDeleteView.as_view(),
name='notam_delete'),
path('cleanup/', cleanup, name='cleanup'),
]

Az urlpatterns változó egy listába foglalja az apphoz tartozó útvonalakat. A path függvény
3 felhasznált argumentum a következő:

```
• route – a relatív URL, amelyen keresztül a View elérhető. A relációs jelek között
látható változókat a router kulcsszó argumentumként továbbítja a View felé
• view – funkcionális view esetén maga a függvény, osztályalapú view esetén pedig
az as_view metódust kell definiálni
• name – az útvonal elnevezésére szolgál.
```
### Adatbázis

#### Adatbázis táblák létrehozása

A táblák létrehozásáért a Django ORM felel, ami egy beépített kapocs a back-end és az
adatbázis között. A táblákat Python classok definiálásával lehet létrehozni egy migrációt
követően.


#### Runway tábla

class **Runway** (models.Model):
airport = models.CharField(max_length= 4 )
designator = models.CharField(max_length= 3 )
category = models.CharField(max_length= 5 )

Futópályák mezői:

```
• airport: Repülőtér 4 betűs ICAO kódja
• designator: Futópálya jele
• category: Műszeres megközelítés kategóriája
```
#### Airport tábla

class **Airport** (models.Model):
icao = models.CharField(max_length= 4 )
purpose = models.CharField(max_length= 4 )^

Repülőterek mezői:

```
• icao: Repülőtér 4 betűs ICAO kódja
• purpose: Repülőtér besorolása (bázis, desztináció)
```
#### Notam tábla

class **Notam** (models.Model):
notam_id = models.CharField(max_length= 15 )
airport = models.CharField(max_length= 4 )
qcode = models.CharField(max_length= 4 )
message = models.CharField(max_length= 511 )
startdate = models.DateTimeField()
enddate = models.DateTimeField()
comment = models.CharField(max_length= 511 , default="")^

NOTAMok mezői:

```
• notam_id: Egyedi azonosítószám
• airport: Repülőtér 4 betűs ICAO kódja
• qcode: NOTAM 4 betűs kategória kódja
```

```
• message: Teljes szabad szöveg
• startdate: Érvényesség kezdete
• enddate: Érvényesség vége
• comment: Teljes szövegből kiszűrt lényegi információ
```
#### User tábla

A felhasználók adatainak tárolására használ tábla beépítve érkezik a Django
keretrendszerrel.

```
Mezők:
• username: A felhasználónév
• password: A felhasználó hashelt jelszava
• first_name: Keresztnév, opcionális
• last_name: Vezetéknév, opcionális
• email: Email cím, opcionális
• groups: Lehetőség van felhasználó csoportok létrehozására, ez a mező tárolja, mely
csoportokhoz tartozik a felhasználó. Ebben a projektben nem használt
• user_permissions: Jellemzően az adatbázis bejegyzések módosításához szükséges
engedélyeket tárolja
• is_staff: Boolean, az admin oldal elérését teszi lehetővé
• is_active: Boolean, annyit takar, hogy aktívnak tekinthető-e a felhasználó. Ajánlott
gyakorlat törlés helyett inaktívra állítani a felhasználót
• is_superuser: Boolean, igaz érték esetén automatikusan megkap minden engedélyt
a felhasználó
• last_login: Utolsó bejelentkezés ideje
• date_joined: Felhasználó létrehozásának ideje
```

### Autentikáció

A Django beépített megoldásokat kínál a felhasználó kezelésére, így a regisztráció
kivételével csupán templateket kellet létrehozni, és egy URL-hez kötni a beépített View-kat.

#### Felhasználó regisztráció

```
Form létrehozása
```
class **SignUpForm** (UserCreationForm):
email = forms.EmailField(max_length= 254 , help_text='Required.
Inform a valid email address.')
class **Meta** :
model = User
fields = ('username', 'email', 'password1', 'password2',
)^

A beépített UserCreationForm kibővítésével lett létrehozva a form. A model változóval a
létrehozandó modelt definiálhatjuk. A fields változó tartalmazza az összes megjelenítendő
input mezőt.

```
SignUpView létrehozása
```
class **SignUpView** (CreateView):
form_class = SignUpForm
success_url = reverse_lazy('login')
template_name = 'signup.html'

A Django adta lehetőségek kihasználása érdekében osztály alapú View-t választottam a
regisztrációnak. A CreateView új adatbázis sor létrehozására alkalmas beépített osztály. a
form_class változóval hozzácsatolhatjuk a már korábban létrehozott formot, ezzel együtt
pedig a User modelt is. A success_url az átirányítási lapot adja meg, a template_name pedig
a megjelenítendő html fájlt.

A User model használata lehetőséget ad a későbbiekben a különböző lapok elérésének
korlátozására két dekoráció, a login_required és a staff_member_required használatával.
Előbbi a bejelentkezett felhasználókra korlátozza az oldal láthatóságát, utóbbi a személyzet
joggal rendelkező felhasználókra.


### Notam model CRUD műveletei

#### Create Notam

```
Új NOTAM létrehozásához először szükség van egy formra:
```
```
class NotamForm (forms.ModelForm):
class Meta :
model = Notam
fields = "__all__"^
Ezt a formot bele kell illeszteni a hozzá tartozó View-ba.
```
@method_decorator(staff_member_required, name='dispatch')
class **NotamCreateView** (CreateView):
model = Notam
form_class = NotamForm
success_url = reverse_lazy("notam_list")
template_name = "notam_create.html"^
Ennél a View-nál megjelenik a staff_member_required dekorátor, mivel nem szeretném, ha
egy mezei felhasználó is módosítani tudná az adatbázis tartalmát. Ezen kívül minden már
változó jelentése már ismert. A form megjelenítése a html fájlban a következő módon néz ki:

A kapcsos zárójelek a Django Template Language sajátosságai. A dupla kapcsos zárójel
változók megjelenítésére használatos, a százalék jellel kiegészített pedig különböző
kifejezések alkalmazására. Jelen példában az url és a csrf_token parancsok láthatóak. Előbbi
az utána megadott név alapján visszakeresi a címet, ahova a POST requestet küldeni fogja,
utóbbi pedig cross-site request forgery támadásoktól védi az oldalt, ami nélkül formot nem is
hajlandó renderelni a keretrendszer.


#### Retrieve Notam

Az adatbázis tartalmának megjelenítésére a beépített ListView nyújt segítséget. A
dekorátor ezúttal a login_required lesz, ez a lap ugyanis nem ad lehetőséget adatok
módosítására.

@method_decorator(login_required, name='dispatch')
class **NotamListView** (ListView):
model = Notam
template_name = 'notams.html'^
A lista html megjelenítése lent látható. A Template Language lehetővé teszi for loopok
használatát html-ben.


#### Update Notam

Ehhez a művelethez nincs szükség új form létrehozására, felhasználható a már meglévő
NotamForm.

```
@method_decorator(staff_member_required, name='dispatch')
class NotamUpdateView (UpdateView):
model = Notam
form_class = NotamForm
template_name = "notam_update.html"
success_url = reverse_lazy("notam_list")^
```
#### Delete Notam

```
Hasonlóan egyszerű az adatbázisból való eltávolítás megoldása is:
```
```
@method_decorator(staff_member_required, name='dispatch')
class NotamDeleteView (DeleteView):
model = Notam
success_url = reverse_lazy("notam_list")^
```
### NOTAM feldolgozás

#### Célok meghatározása

A program feladatának meghatározásakor az első pont a számunkra fontos NOTAMok
adatbázisba rendezése volt. A NOTAMok forrása az ICAO által szolgáltatott API Data
Service, ahonnan JSON formátumban érkeznek az adatok. A második pont az információ
szűrés az időnként kifejezetten hosszú szöveg tartalmában. Az alábbi képen egy budapesti
műszeres megközelítési eljárás ideiglenes használaton kívül helyezéséről szóló NOTAM
látható.

Ez egy rövid és jól értelmezhető NOTAM, de vannak 1000 karakteres verziók is. Az első
sor legelején található a NOTAM id (A1216/21). Az adatbázisban ez az országkód


kiegészítésével kerül elmentésre, ez esetben LH A1216/21 lesz a mező tartalma. Az első
sorból még egy fontos részletet kell kiemelni, amit a qcode. A fenti példa esetében ez
QICAS. A második és harmadik karakter azt közli, hogy mire vonatkozik a NOTAM, a
negyedik és ötödik pedig azt, hogy milyen változás lép életbe.

#### Regexek

A regexek megadott mintákat keresnek a szövegben, és ha egyezést talál, változóba mentik
az eredményt. A NOTAMok esetében a feladat az volt, hogy minden olyan qcode-hoz,
amilyen NOTAMokat az adatbázisba kell menteni, legyen meghatározva egy regex minta. Ha
a qcode-okra szeretnénk mintát felállítani, az alábbi kódot használhatnánk.

QCODES = r"\/Q[A-Z] **{4}** \/"
A fenti minta egyezni fog minden olyan szövegrészlettel, ami törtvonalak közötti, Q-val
kezdődő 5 nagybetűből áll. Mivel a NOTAMok szövegezése még azonos qcode-ok esetén is
nagyban különbözik, így a használt regexek ennél jelentősen hosszabbak.

MRLC = r"(?:RWY|RUNWAY) *[0-9] **{2}** [LRC]*(?:\/[0-9] **{2}** [LRC]*)*
*(?:CLOSED|CLSD)"

FAAH = STAH = FALC = ATCA = SPAH = ACAH = AECA = r"(?:(?:[A-
Z] **{3}** )|(?:[0-9] **{2}** ))[\-\/A-Z0-9 ,]{0,24}:*(?: [0- 9 \-]*)*(?:[
\n,]*[0-9: ]{4,5}-[0-9: ]{4,5})+"

PIAU = r"(?:ILS[A-Z0-9 ]*(?:RWY[0-9RLC ]{2,3})*(?: CAT
[I]{1,3}[AB]*)*[A-Z0-9,\/ ]*(?:SUSPENDED|NOT AVBL))|(?:NOT
AVAILABLE:\nILS[A-Z0-9 ]*RWY[0-9RLC ]{2,3}(?: CAT [I]{1,3}[AB]*)*)"

ICCT = r"(?:ILS|INSTRUMENT LANDING SYSTEM)[\) A-Z]* RWY[ 0-9RLC]*
(?:OPERATING )*(?:ON TEST)*"

ICAS = ISAS = IGAS = ILAS = IUAS = r"(?:ILS\)*(?:[ A-Z] **{4}** )* (?:GP
)*(?:LOC )*(?:CAT [I\/AB]* )*(?:FOR )*RWY[ 0-9RLC\/]*)|(?:RWY[ 0-
9RCL:\/]*ILS)"

FIAU = r".*"


#### Segéd függvények

A segéd függvények a regexeket veszik alkalmazásba, egyrészt minden qcode-hoz tartozik
egy függvény, másrészt egy-egy függvény feladata a NOTAMok periódusának meghatározása
ill. a teljes üzenet kivonása (a példa NOTAMban az E) és a CREATED közötti szöveg).

def MRLC(string):
pattern = getattr(regex, "MRLC")
result = re.findall(pattern, string)
return ", ".join(result)^
Egyszerűbb qcode-okhoz tartozó függvények a fenti sablonon alapszanak. Ezek esetében a
regex önmagában elég a kívánt eredmény eléréséhez, nincs szükség korrigálásra. A pattern
változóhoz hozzárendeli a regex modulból az MRLC változó értékét. A re.findall függvény
összehasonlítja a pattern stringet a függvény argumentumként definiált stringgel, és minden
egyező szövegrészletet listába gyűjt. A függvény végül a lista tartalmát egy stringként, a
tagokat vesszővel elválasztva visszatéríti.

Vannak azonban olyan qcode-ok, ahol a kreált regexek önmagukban zajjal járnak. Ilyen
például a repülőterek nyitvatartását vizsgáló regexek. Ezeket a zajokat a segéd függvény
hivatott fixálni. Az ellenőrzéshez még két változó lett létrehozva.

DAYS = ["MON", "TUE", "WED", "THU", "FRI", "SAT", "SUN"]
NUM = "0123456789"^

```
def FAAH(string):
pattern = getattr(regex, "FAAH")
result = re.findall(pattern, string)
if not result:
return ""
text = ", ".join(result)
if text[: 3 ] in DAYS or text[ 0 ] in NUM:
return f"AD OPEN { text } "
while text[: 3 ] not in DAYS and text[ 0 ] not in NUM:
text = text[ 1 :]
return f"AD OPEN { text } "^
```

Ha a re.findall függvény nem talál egyezést, akkor a visszatérés egy üres string lesz. Ha
talált egyezést, akkor egy stringgé alakítja a lista tartalmát, majd megvizsgálja, hogy a string a
hét valamely nevével, vagy számmal kezdődik, a nyitvatartást ugyanis napokra vagy dátumra
is kiadhatják.

Ha más szavakkal kezdődik a szöveg, addig törli az elejét, amíg meg nem valósul a kívánt
eredmény. Végül az elejére tűzi az „AD OPEN” szavakat, és visszatéríti a stringet.

Más megközelítést igényel a műszeres megközelítés korlátozására vonatkozó regex
kezelése, azon belül is kétféle NOTAMot kell vizsgálni. Az egyik a tesztelés alatt álló
műszerekről ad információt, és a cél, hogy „ON TEST” szóra végződjön.

def ICCT(string):
pattern = getattr(regex, "ICCT")
result = re.findall(pattern, string)
if **not** result:
return ""
text = ", ".join(result)
if **not** text.endswith("ON TEST"):
text += " ON TEST"
return text^
A másik korrigálni való, amelyik a működésen kívüli műszereket taglalja, ezek ugyanis
gyakran a működésen kívüli rendszer frekvenciáját is közlik. Ez azonban számunkra nem
lényegi információ, viszont a regexeket megzavarja egy kissé, és a csatorna frekvenciát is
megtalálja a vesszőig. A függvény ezért leellenőrzi, hogy az adott találat esetén is ez történt-
e, és ha igen, eltávolítja a nem kívánt számokat.


def ICAS(string):
pattern = getattr(regex, "ICAS")
result = re.findall(pattern, string)
if **not** result:
return ""
text = ", ".join(result)
if all(c **in** NUM for c **in** text[- 3 :]):
text = text[:- 4 ]
return f' **{** text **}** NOT AVBL'^
Fontos információt tartogat még egy NOTAM D) szekciója, amely az érvényességi időn
belüli periódusokat jelzi. Amennyiben meghatározzák ezt, a változások nem a NOTAM teljes
hatálya alatt aktívak, hanem csak a megadott periódusokban.

D_PERIOD = r"D\)[ \-A-Z0- 9 \n]*E\)"
def periods(string):
pattern = getattr(regex, "D_PERIOD")
result = re.findall(pattern, string)
return result[ 0 ][ 2 :- 2 ].strip() if result else ""^
D) szekció jelenléte esetén visszatéríti annak tartalmát, ellenkező esetben üres stringet ad.
Az utolsó segéd függvény feladata kezelni a többi, regexeket feldolgozó függvényt, és ennek
a visszatérített értéke kerül az adatbázis Notam táblájának comment mezejébe.


```
def comment(string, qcode):
period = periods(string)
content = getattr(sys.modules[__name__], qcode)(string)
```
```
if "BTN" in string and "BTN" not in period and qcode == "MRLC":
return ""
if qcode == "PIAU" and period and not content:
return ""
```
return f" **{** period **} {** content **}** " if period else
f" **{** content **}** ".replace(" **\n** ", "")

A függvény argumentumai a NOTAM teljes szövegtartalma, valamint a qcode. Lefuttatja a
periódus kereső függvényt, illetve a qcode-dal megegyező elnevezésű függvényt. A comment
függvény részeként lett meghatározva még két kivétel, melyek esetén üres függvényt tér
vissza. Az egyik esetben a futópálya zárás nem a pálya teljes hosszára érvényes, így az
korlátozottan elérhető, ezeket a NOTAMokat nem szeretnénk pályazárként feltűntetni. A
másik eset a PIAU qcode-dal rendelkező NOTAMokra vonatkozik, melyek többféle érkezési
eljárás felfüggesztését jelzik, azonban jelen esetben csak az ILS megközelítések az érdekesek.
Ezért a más megközelítések esetén a content változó üres marad, de a period változónak lehet
értéke. Ezt azonban nem kell az adatbázisba menteni, továbbá content hiányában érthetetlen is
lenne.

#### Airport tábla feltöltése

A repülőtér lista forrása egy CSV fájl. A fájl az AirportFormon keresztül kerül feltöltésre,
az egyetlen input mező egy FileField lesz. A data_validation metódus a fájl kiterjesztését
hivatott ellenőrizni, a process_csv pedig előbb beolvassa, majd dict típusú adatszerkezetként
menti a fájt adatait. Ezután egy listába készíti az adatbázisba mentendő adatokat, és a
bulk_create metódussal egy hívásban hozza létre az adatbázis sorokat.


```
class AirportForm (forms.Form):
file_upload = forms.FileField()
```
```
def data_validation(self):
file = self.cleaned_data["file_upload"]
```
```
if not file.name.endswith(".csv"):
raise forms.ValidationError("File type not supported!")
return file
```
def process_csv(self):
csv_file =
io.TextIOWrapper(self.cleaned_data["file_upload"].file)
reader = csv.DictReader(csv_file)

```
airports = [Airport(
icao = row["ICAO"],
purpose = row["Type"]
) for row in reader if row["Type"] in ["BASE", "DEST"]]
```
Airport.objects.bulk_create(airports)^
A form megjelenítését végző view a beépített FormView kiterjesztésével hozható létre. A
kitöltendő változók nem különböznek a CRUD operációt végző view-kban használtaktól. A
form definiálásakor is csak egy különbség van, a fájl feltöltésekor meg kell adni a megfelelő
értéket az enctype attribútumnak.

```
@method_decorator(staff_member_required, name='dispatch')
class AirportView (FormView):
template_name = "airports_upload.html"
form_class = AirportForm
success_url = "/"^
```

#### API hívása

A NOTAMok lehívása és feldolgozása is a call view feladata. A view két módon érhető el,
egyenesen a call URL hívásával, vagy a NotamSelectView formján keresztül.

```
class NotamSelectForm (forms.Form):
choices = enumerate(["BASE", "BASE+DEST", "SPECIFIC"])
locations = forms.TypedChoiceField(choices=choices)
remark = forms.CharField(required=False)^
A NotamSelectForm 2 input mezőt hoz létre:
• locations: 3 választási lehetőség, bázisok, bázisok és desztinációk, vagy specifikus
• remark: szabad szöveges input mező, specifikus opció esetén az itt felsorolt
repterekre fogja lekérni a NOTAMokat.
```
@method_decorator(staff_member_required, name='dispatch')
class **NotamSelectView** (FormView):
template_name = "notam_select.html"
form_class = NotamSelectForm
success_url = reverse_lazy("notam_select")^
A NotamSelectView azonos minta szerint lett létrehozva, mint a korábbi, formot
megjelenítő view-k.

A html form action attribútuma a call view-ra mutat, submit esetén tehát egy POST
requesttel kezdeményezzük az API hívást. Az API cím 3 kötelező paraméterrel rendelkezik:


```
• api_key: érvényes kulcs nélkül nem kapunk adatokat
• format: CSV vagy JSON
• locations: egy, vagy több ICAO kód vesszővel elválasztva
```
API_ADDRESS =
f"https://applications.icao.int/dataservices/api/notams-
list?api_key= **{** API_KEY **}** &format=json&locations="

```
A call view első feladata meghatározni, mi lesz a locations paraméter értéke.
```
if request.method == "GET":
locations = [airport.icao for airport **in**
Airport.objects.filter(purpose__in=["BASE", "DEST"])]
elif request.method == "POST":
if request.POST["locations"] == "0":
locations = [airport.icao for airport **in**
Airport.objects.filter(purpose="BASE")]
elif request.POST["locations"] == "1":
locations = [airport.icao for airport **in**
Airport.objects.filter(purpose__in=["BASE", "DEST"])]
elif request.POST["locations"] == "2":
locations = request.POST["remark"].split()

GET request esetén alapértelmezetten minden bázisra és desztinációra lekéri a
NOTAMokat. POST request esetén a kérés locations értékét veszi alapul a paraméter
megadásának.

response = requests.get(f" **{** API_ADDRESS **}{** ','.join(locations) **}** ")
notamdata = response.json()^

A request.get metódus elküldi a kérést az API szolgálat felé, majd a visszakapott JSON
formátumú adatot a json metódus átkonvertálja Python által kezelhető objektummá, majd
elmenti a notamdata változóba.

{
"_id": "60718282f2b9a3f8f8392eac",
"id": "A1320/21",
"entity": "AT",
"status": "CA",
"Qcode": "ATCA",
"Area": "ATM",


"SubArea": "Airspace organization",
"Condition": "Changes",
"Subject": "Terminal control area",
"Modifier": "Activated",
"message": "BEAUVAIS CTR, TMA AND AD CTL HOURS OF ACT :\n-MON-SAT :
0500 - 2100,\n-SUN : 0400-2100,\n-EXTENSION FOR ANY SKED COMMERCIAL
FLIGHT,\n-ACTUAL ACTIVITY AVBL ON ATIS.\nCREATED: 30 Mar 2021 07:44:00
\nSOURCE: EUECYIYN",
"startdate": "2021- 03 - 30T07:43:00.000Z",
"enddate": "2021- 06 - 28T23:59:00.000Z",
"all": "A1320/21 NOTAMR A1246/21\nQ)
LFFF/QATCA/IV/NBO/AE/000/085/4934N00209E026\nA) LFOB B) 2103300743 C)
2106282359 \nE) BEAUVAIS CTR, TMA AND AD CTL HOURS OF ACT :\n-MON-SAT :
0500 - 2100,\n-SUN : 0400-2100,\n-EXTENSION FOR ANY SKED COMMERCIAL
FLIGHT,\n-ACTUAL ACTIVITY AVBL ON ATIS.\nCREATED: 30 Mar 2021 07:44:00
\nSOURCE: EUECYIYN",
"location": "LFOB",
"isICAO": true,
"Created": "2021- 03 - 30T07:44:00.000Z",
"key": "A1320/21-LFOB",
"type": "airport",
"quality": {
"fmt_01": 1,
"len_01": 1,
"dur_01": 1,
"dur_03": 0,
"qcd_02": 1,
"qcd_03": 1,
"prp_01": 1,
"prp_02": 1,
"prp_03": 0,
"jar_01": 0,
"jars": [
"ACT",
"AVBL",
"CTL"
],
"score": 70
},
"StateCode": "FRA",
"StateName": "France"


}

A fent látható példa egy Beauvais repterére kiadott NOTAM adatait tartalmazza JSON
formátumban.

```
notam_ids = [notam.notam_id for notam in Notam.objects.all()]
```
notams = [Notam(
notam_id = f" **{** data['location'][ 0 : 2 ] **} {** data['id'] **}** ",
airport = data["location"],
qcode = data["Qcode"],
message = data["message"] if len(data["message"]) < 511 else
data["message"][: 511 ],
startdate = datetime.datetime(
int(data["startdate"][ 0 : 4 ]),
int(data["startdate"][ 5 : 7 ]),
int(data["startdate"][ 8 : 10 ]),
int(data["startdate"][ 11 : 13 ]),
int(data["startdate"][ 14 : 16 ])),
enddate = datetime.datetime(
int(data["enddate"][ 0 : 4 ]),
int(data["enddate"][ 5 : 7 ]),
int(data["enddate"][ 8 : 10 ]),
int(data["enddate"][ 11 : 13 ]),
int(data["enddate"][ 14 : 16 ])),
comment = util.comment(data["message"], data["Qcode"])
) for data **in** notamdata if f" **{** data['location'][ 0 : 2 ] **}
{** data['id'] **}** " **not in** notam_ids **and** data["Qcode"] **in**
["MRLC", "FAAH", "STAH", "FALC",
"ATCA", "SPAH", "ACAH", "AECA",
"PIAU", "ICCT", "ICAS", "ISAS",
"IGAS", "ILAS", "IUAS", "FIAU"]]^
A notam_ids változó a függvény hívásakor adatbázisban szereplő összes NOTAM
hivatalos (nem az adatbázis által adott, hanem a kiadó hatóság által meghatározott) id-ját
tartalmazza. A notams lista Notam objektumokat hoz létre a kapott JSON adatok megfelelő


rovatai alapján, valamint a comment segéd függvény alkalmazásával. A listába azok a
NOTAMok kerülnek be, amelyek az előre megadott qcode-ok egyikével rendelkeznek, és
még nem szerepelnek az adatbázisban. A datetime könyvtár segít átalakítani a NOTAMban
használt dátumformátumot a Python által kezelhetővé.

```
Notam.objects.bulk_create(notams)
```
return redirect(reverse_lazy("notam_list"))^
A bulk_create létrehozza a bejegyzéseket az összes, notams listában jelenlévő NOTAMra.
A view végül átirányít az összes NOTAMot listázó oldalra.

#### Jelentés generálása

A reporting app egyetlen view segítségével készít jelentést az adatbázisban tárolt
NOTAMokról. POST request esetén emailben elküldi a jelentést az előre beállított
címzettnek. A jelentést a request message paramétere tartalmazza. A send_mail függvény
szintént a Django beépített elemeinek egyik eleme.

def verbose_report(request):
if request.method == "POST":
subject = "verbose report"
message = request.POST["message"].replace("<br>", " **\n** ")
email_from = settings.EMAIL_HOST_USER
recipient_list = ["botond@roff.hu"]
send_mail(subject, message, email_from, recipient_list)
return redirect(reverse_lazy("notam_list"))^
GET request esetén meghatározza az időintervallumot, amire készíti a jelentést, ez a
request időpontjától számított 24 óra. Az adatbázisból lekéri az ebben az időszakban érvényes
NOTAMokat, valamit a repülőtereket.

else:
start = datetime.datetime.now()
end = start + datetime.timedelta(hours=24)
notams = Notam.objects.filter(startdate__lt=end)
airports = {airport.icao: airport.purpose for airport **in**
Airport.objects.all()}


Definiál két dict adatszerkezetet, egyet a bázisoknak, egyet a desztinációknak. For ciklus
megvizsgálja a notams lista összes elemét a következő módon. A text változóhoz hozzárendeli
a base szótárban a reptér kulcshoz tartozó értéket, illetve ha még nincs a reptérnek értéke, egy
üres stringet, majd ehhez hozzáadja az aktuális NOTAM comment mező tartalmát.
Leellenőrzi, hogy a kijelölt 24 óra teljes hosszában érvényes-e a vizsgált NOTAM, és ha
nem, kiegészíti a kezdő és/vagy záró időponttal. Majd frissíti a szótárban a repülőtérhez
hozzárendelt értéket.

base = {}
for notam **in** sorted(notams, key=lambda x: x.airport):
text = f" **{** base.get(notam.airport,
'') **}\n\t\t{** notam.comment **}** ".strip()
if start < notam.startdate.replace(tzinfo=None) < end:
text += f" FROM **{** str(notam.startdate)[ 8 : 16 ] **}** "
if start < notam.enddate.replace(tzinfo=None) < end:
text += f" TILL **{** str(notam.enddate)[ 8 : 16 ] **}** "
if airports.get(notam.airport, "") == "BASE" **and**
notam.comment.strip():
base[notam.airport] = text
dest = {}
for notam **in** sorted(notams, key=lambda x: x.airport):
text = f" **{** dest.get(notam.airport,
'') **}\n\t\t{** notam.comment **}** ".strip()
if start < notam.startdate.replace(tzinfo=None) < end:
text += f" FROM **{** str(notam.startdate)[ 8 : 16 ] **}** "
if start < notam.enddate.replace(tzinfo=None) < end:
text += f" TILL **{** str(notam.enddate)[ 8 : 16 ] **}** "
if airports.get(notam.airport, "") == "DEST" **and**
notam.comment.strip():
dest[notam.airport] = text

A view végül egy stringbe rendezi a két szótár tartalmát, és a render függvény context
argumentumához adja.


message = "BASE **\n** "
for airport, notam **in** base.items():
message += f" **{** airport **}\t{** notam **}\n** "
message += " **\n** DEST **\n** "
for airport, notam **in** dest.items():
message += f" **{** airport **}\t{** notam **}\n** "

context = {
"message": message
}
return render(request, "report_preview.html",
context=context)

### Tesztdokumentáció

#### Kompatibilitás

Tesztelt böngészők:
• Google Chrome
• Internet Explorer
• Microsoft Edge
A honlap fejlesztése végig Chrome használata mellett történt. Az Explorer szemmel is
láthatóan elmaradt teljesítményben a többi böngészőtől, főleg kulcsszavas kereséskor lehetett
lassulásokat tapasztalni. Emellett kisebb-nagyobb különbségekkel találkoztam a stíluslap
feldolgozásában is (filter property nem működik Exploreren, smooth-scrolling csak Chrome-
on teljesül).


## Felhasználói dokumentáció

### Oldal elérése

A honlap a [http://notam-process.herokuapp.com/](http://notam-process.herokuapp.com/) címen érhető el. Az applikáció irodai
használatot szem előtt tartva lett tervezve, így számítógépen ajánlott használni, nem
mobiltelefonon vagy tableten. A böngészők közül a Chrome alkalmazása javasolt.

```
A tesztelés érdekében létrehozott felhasználó:
Username: test
Password: testpassword
```
### Fiók kezelés

#### Regisztráció

Regisztrálni a navigációs sáv jobb szélén található „Signup” gombra kattintva lehet. A
megjelenő oldalon az adatok kitöltésével elkészül az új felhasználó. Felhasználónév
maximum 150 karakter lehet, betűkből (kis- és nagybetű, ékezetes is lehet), számokból illetve
„_”, „@”, „+” „.” és „-„ karakterekből állhat. A jelszó legalább 8 karakterből kell álljon, nem
hasonlíthat a felhasználónévre és az email címre, nem állhat kizárólag számokból, és nem
lehet túl egyszerű (pl. a password szót jelszóként nem fogja elfogadni). Az email cím
megadása még nem implementált funkciók miatt szükséges. A regisztráció aktiválására nincs
szükség, azonnal be lehet jelentkezni.

#### Bejelentkezés

A navigációs sávon található „Log in” gombra kattintva megjelenik a bejelentkezési oldal.
Felhasználónév és jelszó megadása után átirányít a főoldalra, és elérhetővé válnak a
bejelentkezéshez kötött oldalak.

#### Kijelentkezés

Bejelentkezett felhasználó esetén a navigációs sávon megjelenik a felhasználó név, és a
„Log Out” gomb. Utóbbira kattintva kijelentkezünk a felhasználói fiókból.


### Bejelentkezéshez kötött oldalak használata

#### NOTAMok listázása

Az adatbázisban megtalálható NOTAMok listázásához a „List notams” feliratú kártyára
kell kattintanunk. Az átirányított oldalon láthatjuk a teljes listát, illetve szűrhetünk kulcsszó,
repülőtér, qcode valamint érvényességi idő szerint. Minden szűrési feltétel teljesül a „Filter”
gombra kattintva, így ki tudjuk szűrni például az összes, ezen a héten érvényes futópálya
zárást egy tetszőleges reptéren. Kulcsszóra a „Search for keywords” szöveggel kitöltött
mezőben tudunk keresni. A „Reset” gombra kattintva minden szűrési beállítás alapértelmezett
állapotba kerül, és a teljes lista jelenik meg.


A NOTAMok kártyaként jelennek meg, felső részén a qcode-hoz tartozó kép, alsó részén a
comment mező tartalma látható. A kép bal felső sarkában a NOTAMhoz vonatkozó repülőtér
ICAO kódja, jobb felső sarkában a qcode olvasható.

```
A képre kattintva felugranak a NOTAM további részletei:
• comment: vastag betűkkel olvasható az ablak felső részén
• full message: a NOTAM teljes szövege
• valid from/till: érvényességi idő kezdete és vége
• update gomb: NOTAM módosítása, csak staff felhasználók számára elérhető
```

```
• delete gomb: NOTAM törlése, csak staff felhasználók számára elérhető
```
```
X-re kattintva eltűnik.
```
#### Jelentés generálása

A szűrők feletti gombsor jobb szélső gombjára („Generate report”) kattintva megjelenik
minden, a következő 24 órában érvényes NOTAM, bázisokra és desztinációkra bontva. A
felette található „Send report” gombbal a listát elküldi emailben az előre beállított email
címre.

### Korlátozott elérésű oldalak használata

#### Engedélyek

A korlátozott elérésű oldalak eléréséhez staff státuszú fiókra van szükség. Ezt a státuszt
egy, a megfelelő engedélyekkel rendelkező felhasználó tudja megadni az admin felületen.
Ennek oka, hogy alapvetően egy „zártkörűen” használt alkalmazás készítése volt a terv, ezért
az adatbázis bejegyzések módosítására személyesen kiválasztott felhasználók lehetnek csak
képesek. A feldolgozott NOTAMok megtekintése viszont hasznos lehet mások számára is,
ezért az egy alap felhasználói fiókkal is elérhető.

#### Admin felület

A Django admin felület lehetőséget ad az adatbázisban tárolt adatok létrehozására,
módosítására, törlésére, beleértve a felhasználók és azok engedélyeit is. Ezen műveletek egy
része a honlapon keresztül is elérhető, azonban bizonyos esetekben, pl. teszteléskor


kényelmesebb megoldás a kibővített funkciókkal rendelkező admin felületen keresztül
elvégezni őket. Az admin felület relatív címe „/admin/”.

#### NOTAMok letöltése, feldolgozása

A főoldalról kiindulva, a „Process new notams” feliratú kártyára kattintva eljutunk a
repülőterek kiválasztását lehetővé tevő formhoz. A „locations” címke melletti lenyíló
menüből 3 opció közül választhatunk:

• BASE – kizárólag a bázisokra kérjük le a legutóbbi frissítés óta kiadott
NOTAMokat
• BASE+DEST – bázisokra és desztinációkra is
• SPECIFIC – egyedi reptereket szeretnénk megadni
Utóbbi választásakor, a „remark” mezőben adhatjuk meg szóközzel elválasztva a kívánt
repterek ICAO kódját. A „Process” gombra kattintva elküldjük a kérést az API szolgáltató
felé, majd néhány másodperc után a szerver átirányít NOTAM lista oldalra. A lista oldal
tetején megtalálható „Process new Notams” gomb az alapértelmezett (BASE+DEST)
beállítással küldi el a kérést az API számára.

#### NOTAM hozzáadása

Az oldal tetején megtalálható gombsorban az „Add new Notam” klikkelésével adhatunk
hozzá új NOTAMot. A megjelenő formon a kitölthetjük a kívánt adatbázis mezőket, majd a
„Create Notam” gomb megnyomásával elmenthetjük azt az adatbázisban.


#### NOTAM módosítása

A NOTAM kártya képére kattintva megnyitjuk a NOTAM részletes adatait. Ezen ablak
alján található „Update” gomb megnyomásával az létrehozáshoz hasonló formhoz jutunk el.
A kívánt módosítások elvégzése után az „Update Notam” gomb elmenti a változtatásokat.

#### NOTAM törlése

```
Az Update gomb melletti Delete gombbal eltávolíthatjuk a NOTAMot az adatbázisból.
```
#### Lejárt érvényességű NOTAMok eltávolítása

A lista oldal tetején található „Cleanup” gomb törli az összes NOTAMot az adatbázisból,
amely érvényessége előző nappal bezárólag befejeződött.


## Összefoglalás

Amit előzetesen kitűztem a záródolgozat írásának idejéig, azt nagyrészt teljesíti a program.
Két javításra szoruló problémáról tudok jelenleg. A NOTAMok szövegezésének különbségei
miatt néhány NOTAM nem akad fenn a regexek szűrőjén, vagy kiszűr belőle adatokat, de
nem minden fontos információt. Ezért a regex mintákat folyamatosan ellenőriznem kell a
megfelelő működéshez. A másik hiba inkább esztétikai. A jelentés megjelenítésén mind a
honlapon, mind az emailben elküldött eredményen dolgoznom kell a jobb olvashatóság
érdekében. Ezen kívül a honlap kinézete is javításra szorul, mind a színek kontrasztosabbá
tétele, mind a reszponzivitás fejlesztése napirenden van.

Jövőbeni tervek között szerepel a honlap által ellátott feladatok ütemezése: naponta kétszer
automatikusan frissíteni az adatbázist, majd elküldeni a jelentést. A regex minták folyamatos
felülvizsgálatán túl a vizsgált qcode-ok bővítése is tervben van, de ezeken kívül is szeretném
további funkciókkal ellátni az oldalt a jövőben.

## Irodalomjegyzék

```
• https://docs.djangoproject.com/
• https://www.fullstackpython.com/
• https://simpleisbetterthancomplex.com/
• https://www.w3schools.com/
• https://stackoverflow.com/
• https://freefrontend.com/
• https://realpython.com/
• https://css-tricks.com/
```
## Köszönetnyilvánítás

Szeretném megköszönni tanáraimnak, Bak Zsuzsának, Kovács Lászlónak, Bak Barnának,
Vonyigás Istvánnak, Gera Imrének és Pártl Károlynak a két év alatt átadott tudást.


