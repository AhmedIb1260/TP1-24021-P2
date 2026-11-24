# Partie A

## Etape 1
1. L'annotation @WebService fait de EmpruntService un service web. L'annotation @WebFault fait de LivreIndisponibleException une erreur déclarée dans le contrat WSDL.   
2. Les classes de données doivent avoir un constructeur vide parce que Java (avec l'outil JAXB) a besoin de créer d'abord un objet vide automatiquement lorsqu'il reçoit un message XML, avant de remplir ses variables une par une avec les valeurs reçues.

## Etape 2
1. L'URL du service est http://localhost:8080/bib/EmpruntService. Les deux opérations sont emprunterLivre (avec le message d'entrée emprunterLivre, le message de sortie emprunterLivreResponse et la faute LivreIndisponibleException) et rendreLivre (avec le message d'entrée rendreLivre et le message de sortie rendreLivreResponse).
2. Le binding déclare le style document et le use literal. Il s'agit de SOAP 1.1.
3. Le type XSD généré est xs:string pour isbn et pour dateRetour, car ils sont déclarés comme des String dans les classes Java. La règle de format de l'ISBN n'apparaît pas dans le XSD parce que la vérification par expression régulière est écrite à l'intérieur du code de la méthode Java.
4. Non, dans la requête SoapUI, les paramètres idAdherent et isbn ne portent pas de préfixe d'espace de noms, contrairement à la balise parente emprunterLivre. Cela s'explique par le fait que JAX-WS génère par défaut des éléments locaux non qualifiés (elementFormDefault="unqualified").
5. Dans l'onglet Raw, HTTP apparaît tout en haut dans les en-têtes (POST, Content-Type, SOAPAction), XML correspond à tout le texte structuré avec des balises, et SOAP correspond aux balises Envelope, Header et Body. HTTP est le protocole de transport qui achemine les messages sur le réseau, XML est le format universel qui structure les données, SOAP définit l'enveloppe standard des messages et des erreurs, et le WSDL est le contrat que SoapUI lit avant l'envoi pour connaître l'adresse du service, les opérations disponibles et le format attendu.

## Etape 3
1. En SOAP sur HTTP, on utilise toujours la méthode POST parce que la requête transporte une enveloppe XML complète dans le corps de la requête HTTP et que le nom de l'opération appelée se trouve à l'intérieur du Body SOAP plutôt que dans l'URL.   
2. Non, le bloc jeton du Header n'a pas empêché le traitement. En SOAP, les éléments du Header sont optionnels par défaut : comme l'attribut mustUnderstand="1" n'est pas présent, le serveur ignore simplement les en-têtes qu'il ne connaît pas et exécute normalement le contenu du Body.   
3. Le service répond avec une erreur HTTP 500 et une Fault SOAP de type VersionMismatch indiquant que l'espace de noms de l'enveloppe est inattendu. D'après le binding lu à l'étape 2, le service est configuré uniquement pour SOAP 1.1 (http://schemas.xmlsoap.org/wsdl/soap/) et refuse donc l'enveloppe SOAP 1.2.   
4. Les cinq erreurs du message sont :   
- La balise d'ouverture <soap: envelope est mal formée (espace après les deux-points, minuscule sur envelope au lieu de Envelope, et soap:Body placé avant la fermeture > de la balise Envelope, ainsi que des espaces après les deux-points dans les autres balises) : il faut écrire <soap:Envelope ...> proprement avant soap:Header et soap:Body.   
- L'espace de noms xmlns:bib vaut http://supnum.mr/bibliotheque au lieu de http://supnum.mr/bib.   
- La balise fermante  n'a pas de préfixe : il faut écrire </bib:rendreLivre>.   
- Le bloc soap:Header est placé après soap:Body alors qu'il doit obligatoirement être placé avant soap:Body.   
- Le paramètre est nommé bib:numero au lieu de  (mauvais nom de paramètre et préfixe inutile sur un élément local). 
  L'erreur sur bib:numero (au lieu de ) ne déclenche aucune erreur côté serveur une fois les autres erreurs corrigées : JAX-WS / JAXB ignore les balises inconnues lors de la désérialisation et passe null pour numeroEmprunt, ce qui permet à la méthode rendreLivre de s'exécuter et de renvoyer 0 sans erreur (la vérification du numéro n'étant ajoutée qu'à l'étape 4).

## Etape 4
1. Dans les deux cas, le serveur renvoie le code HTTP 500 (Internal Server Error), car en SOAP 1.1 sur HTTP toute Fault utilise le statut 500. Pour le cas 1 (ISBN invalide), le faultcode est S:Client et le faultstring est "ISBN mal formé : 978-2-10". Pour le cas 2 (livre déjà emprunté), le faultcode est S:Server et le faultstring est "Livre indisponible".

2. Dans le cas 2, la balise detail contient l'élément structuré LivreIndisponible (dans l'espace de noms http://supnum.mr/bib) avec les sous-éléments isbn (978-2-10-074444-4) et retourPrevu (2026-12-02). Sa structure est décrite dans la section types du WSDL (plus précisément dans le schéma XSD importé ?xsd=1) et référencée par l'élément fault de l'opération emprunterLivre dans le portType et le binding.

3. JAX-WS a choisi par défaut le code Server (S:Server) pour l'exception personnalisée LivreIndisponibleException. Du point de vue métier, il s'agit plutôt d'une erreur liée à la demande du client (il demande un livre qui n'est pas disponible, et non d'une panne technique du serveur), donc le code Client (ou Sender en SOAP 1.2) aurait été plus pertinent pour signifier qu'il est inutile de renvoyer la même requête immédiatement.

4. Avec mustUnderstand="1", le serveur doit obligatoirement comprendre et traiter l'en-tête bib:jeton ; comme aucun traitement n'est prévu pour cet en-tête côté serveur, JAX-WS rejette immédiatement la requête sans exécuter la méthode et renvoie une Fault avec le faultcode S:MustUnderstand. Avec mustUnderstand="0" (valeur par défaut), l'en-tête est facultatif : le serveur ignore la balise inconnue et exécute normalement le contenu du Body.

5. Contrairement aux Faults SOAP 1.1 reçues, la Fault SOAP 1.2 utilise l'espace de noms http://www.w3.org/2003/05/soap-envelope et remplace les balises par des éléments hiérarchiques et normalisés : faultcode devient soap:Code contenant soap:Value (avec soap:Sender au lieu de Client), faultstring devient soap:Reason contenant soap:Text avec l'attribut obligatoire xml:lang="fr", et detail prend une majuscule pour devenir soap:Detail.

# Partie B

## Etape 5
1. L'opération est emprunterLivre, son message de fault est LivreIndisponibleFault (associé à l'élément bib:LivreIndisponible) et l'action SOAP (soapAction) à envoyer est http://supnum.mr/bib/emprunterLivre.

2. La signature générée dans EmpruntPortType est : public EmpruntType emprunterLivre(String idAdherent, String isbn) throws LivreIndisponibleFault;

3. Le type Java généré pour dateRetour est XMLGregorianCalendar (javax.xml.datatype.XMLGregorianCalendar) car le WSDL définit type="xs:date", alors que dans la partie A dateRetour était un simple String. 

4. Le nom de la classe d'exception générée est LivreIndisponibleFault. Ce nom provient directement de l'attribut name="LivreIndisponibleFault" de la balise wsdl:message dans le WSDL.

## Etape 6
1. Oui, le WSDL publié est bien celui fourni dans src/main/resources/wsdl/EmpruntService.wsdl grâce à l'attribut wsdlLocation, et il contient le motif regex de l'ISBN ainsi que le type xs:date.
2. Non, il n'a pas été nécessaire de modifier le fichier 01-emprunter.xml (seul l'en-tête HTTP SOAPAction a été renseigné avec "http://supnum.mr/bib/emprunterLivre"), car le contrat WSDL utilise le même espace de noms (http://supnum.mr/bib), le même nom d'opération et les mêmes noms de paramètres.

3. Sans @SchemaValidation, l'ISBN 978-2-10 n'est pas refusé car JAX-WS ne vérifie pas automatiquement les restrictions XSD (xs:pattern) par défaut et le code Java ne fait pas de vérification manuelle. Avec @SchemaValidation, JAX-WS valide la requête contre le schéma XSD avant l'exécution de la méthode et renvoie une Fault SOAP S:Client.

4. Si l'on retire wsdlLocation, JAX-WS ignore le fichier WSDL fourni et génère un nouveau contrat à partir des classes Java : on perd alors la restriction xs:pattern sur l'ISBN, l'attribut soapAction redevient vide ("") et le schéma XSD est déporté dans un fichier séparé (?xsd=1).

## Conclusion
1. Deux différences entre le WSDL de la partie A et celui de la partie B : le WSDL de la partie B intègre directement un schéma XSD avec des types stricts (xs:date et IsbnType avec xs:pattern) alors que la partie A utilise un XSD externe où tout est typé en xs:string ; de plus, la partie B définit une soapAction explicite et elementFormDefault="qualified", tandis que la partie A génère une soapAction vide et elementFormDefault="unqualified".

2. Si l'on renomme le champ dateRetour dans le code Java, c'est le projet code-first (partie A) qui voit son contrat WSDL changer automatiquement (en contract-first, le WSDL reste figé et le code ne compile plus tant qu'il ne respecte pas le contrat). Le risque pour les clients est de provoquer des erreurs de désérialisation et de casser les applications existantes.

3. Pour le service Paiement appelé par plusieurs équipes et langages (.NET, PHP, Node.js), il faut choisir l'approche contract-first afin de partager un contrat WSDL/XSD strict et indépendant des technologies. Pour un prototype interne de deux jours, l'approche code-first suffit car elle est beaucoup plus rapide à développer. 