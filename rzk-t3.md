Razvoj zasnovan na komponentama - vežbe 3

Mikroservisi i Spring Cloud Config


Spring Cloud Config

Dependencies

● Conﬁg Server (Spring Cloud Conﬁg)
● Conﬁg Client (Spring Cloud Conﬁg)


Anotacije

Iznad glavne klase aplikacije:
● @EnableConﬁgServer

Zadatak

Mikroservisi i Conﬁg Server


Implementirati sistem koji se sastoji od dva mikroservisa i config servera:


conﬁg-server dobavlja parametre za konekciju na bazu iz lokalnog git repozitorijuma

horse-management-service komunicira sa bazom (parametre dobavlja od conﬁg servera) i ima
sledeće endpoint-ove:

/horses - dobavlja sve konje iz baze

/horses/{idHorse} - dobavlja konja sa datim id-jem iz baze

analytics-processing-service nema pristup bazi, već pomoću RestClient-a poziva endpoint-ove
od horse-management servisa i zatim obrađuje dobijene informacije, pa ima sledeće
endpoint-ove:

/analytics/horses - dobavlja sve konje iz baze

/analytics/count - vraća informaciju o broju konja u bazi

/analytics/name/{idHorse} - vraća ime i nadimak konja sa datim id-jem (potrebno je napraviti
DTO klasu za konja)


