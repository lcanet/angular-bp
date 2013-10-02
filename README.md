AngularJS Best practices
==========

## Structure et organisation du code

### Se fixer des conventions de nommages.
Pour tous les �l�m�nts nomm�s, en JS, CSS et pour les identifieurs sp�cifiques � angular
Par exemple:

* Noms de services: pas de pr�fixe $ (r�serv� � angular), camelCase (ex: ``catalogService``) 
* Noms de classes CSS: tiret-case sans majuscules (``button-primary``)
* ID d'�l�ments: camelCase
* Noms d'�v�nements propag�s sur les scopes : camelCase, en pr�fixant par un domaine (``Map.previewUpdated``)

### S�parer l'application en modules fonctionnels
Utiliser les modules angular pour regrouper un ensemble de composants (directives, controlleurs, filtrer, services) avec une logique
fonctionnelle. Ne pas utiliser de regroupement technique (1 module pour tous les filtres, 1 module pour tous les services etc).

Cette s�paration technique peut (et devrait!) apparaitre en revanche dans l'arborescence de fichiers :

* cartotheque/
	* controllers/
		* controller1.js
		* controller2.js
	* filters/
		* previewFilter.js
	* cartotheque.js
* data/
	* controllers/
		* datactrl1.js
	* filters/
		* dataFilter.js
	* data.js


	
### Pr�fixer les filtres et directives

Afin d'�viter toute collisions d'�l�ments nomm�s dans les templates, il est prudent de pr�fixer toutes les directives et tous les filtres
sp�cifiques � un projet, avec un code propre � l'organisation ou au projet

	<pwet-button-panel label="{{user | pwetCamelize}}"></pwet-button-panel>
	

### Utiliser des modules

Utiliser des modules angular pour s�parer l'application en bloc fonctionnels. Des modules peuvent �galement regrouper des �l�ments
communs pour former une librairie r�utilisable dans plusieurs projets.  

	
## Utilisation des composants angular

### Ne Jamais r�f�rencer le DOM depuis les controllers
Il faut au maximum �viter la manipulation du DOM (objets javascript repr�sentant des fragments de bouts de pages HTML en m�moire) dans les controllers
et les services angular.

Cela permet :

* de ne pas avoir de fuites m�moires (blocs de DOM dynamiques jamais r�f�renc�s directement donc garbageable)
* de mieux d�coupler les diff�rences responsabilit�s
* d'�tre facilement testable

Il faut toujours encapsuler les manipulations du DOM donc depuis les directives. Soit avec les directives fournies dans le framework
(show, ng-class, etc) soit avec des directives persos lorsque cela est n�cessaire, par exemple lors de l'utilisation d'une librairie JS tierce. 
La m�thode ``link`` d'une directive recoit l'�l�ment de DOM (wrapp� dans un objet jquery), que l'on peut ainsi manipuler � sa guise.


### Dans une directive, limiter la manipulation de DOM

Une directive manipulant du DOM doit essayer de se restreindre � l'�l�ment qui lui est pass� (perim�tre d'application de la directive) ainsi qu'� ses
descendants. Lorsque l'on recherche des �l�ments descendants, toujours utiliser la m�thode de recherche jquery ``find`` appliqu�e � l'�l�ment 
racine de la directive pass� � la m�thode `` link`` et qui limitera ainsi les �l�ments � la sous arborescence.

	template: '<div><p class="raviolis"></p></div>',
    link: function(scope, elt, attrs) {
		// will only match raviolis in our directive instance and not all raviolis on screen
	    elt.find('.raviolis').show();
	}

	

### Penser au cycle de vie
Tout scope cr�e par angular peut �galement �tre d�truit, par exemple si l'on d�truit le bout de page associ�. Les scopes d�truits �mettent un
�v�nement ``$destroy`` qui peut �tre �cout� pour effectuer divers traitements :

* Nettoyage de donn�es
* Desenregistrement de timers
* Dans le cas d'une directive, nettoyage d'�v�nements ou d'�l�ments ailleurs que sur le sous-arbre du DOM de la directive.

Un exemple classique de ce dernier cas est l'utilisation d'une directive encapsulant les composants bootstrap `` modal`` ou `` colorpicker``. 
Ces composants sont initialis�s avec un bout de code javascript qui placent un morceau de DOM invisible � la fin du body.

Il faut alors capter l'�v�nement de destruction du scope pour appeler les bonnes m�thodes de nettoyage.

	template: '<div></div>',
    link: function(scope, elt, attrs) {
	    var cp = elt.colorpicker();
		scope.$on('$destroy', function(){
			cp.picker.remove();		// remove the <div class="colorpicker"> at root of document
		});
	}

### Avoir des r�f�rences de mod�les immuables

Deux raisons 

1. avoir une donn�e maitris�e par un service central et accessible en lecture dans les controlleurs et dans les scopes
2. modifier une donn�e dans un scope parent

Pourquoi ? Dans le premier cas, si un service est responsable d'une donn�e, il la pr�sentera comme un membre du service et celle-ci
sera r�cup�r� par le controlleur pour l'exposer dans le scope et la rendre disponible dans la vue :

	function MyController($scope, myService) {
		$scope.model = myService.model;
	}
	
Le code suivant ne sera execut� qu'une seule fois � l'initialisation du controlleur. Si la r�f�rence est modifi�e � un moment ult�rieur 
(chargement asynchrone, ...), le scope aura une ancienne version du model qui ne sera plus bonne. Pas besoin de passer par une gestion
d'�v�nement pour cela : il suffit de renvoyer toujours une m�me r�f�rence immuable et le binding sur des attributs de mod�le se chargera
du reste.

	<span>{{model.firstName}}</span>
	
Dans le service:
	
	service.populateModel = function() {
		service.model.firstName = '...';
		// et pas service.model = {firstName: '...'};
	};

Attention, le raccourci consistant � injecter directement le service dans le scope est franchement � bannir :

	function MyController($scope, myService) {
		$scope.service = myService;
	}
	
	<span>{{service.model.firstName}}</span>
	

Le second cas est une particularit� de la gestion de l'h�ritage des scopes. Imaginons le code suivant:

	<span>{{clicked}}</span><a href="" ng-click="clicked = 'VRAI'">click me</a>
	
Pour �valuer le premier binding ``{{clicked}}``, la variable est d'abord �valu�e dans le scope courant, puis si celle-ci n'est pas d�finie
dans le scope parent et ainsi de suite jusqu'au scope racine. 
Imaginons que dans notre cas la variable soit d�finie dans le scope racine. Le click, ne va pas s'appliquer � la variable {{clicked}} du scope
dans lequel elle est d�finie, mais directement dans le scope courant !

Avant:
	
	rootScope	clicked='FAUX'
	^
	|
	scope		/
	
Apr�s
	
	rootScope	clicked='FAUX'
	^
	|
	scope		clicked='VRAI'
	
D'o� un gros probl�me si d'autres scopes 'frangins' du scope courant utilisent �galement la variable ``clicked``.

La solution consiste � encapsuler les attributs modifi�s dans un objet dont la r�f�rence sera immuable :

	<span>{{wrapper.clicked}}</span><a href="" ng-click="wrapper.clicked = 'VRAI'">click me</a>

Avant:
	
	rootScope	wrapper { clicked='FAUX' }
	^
	|
	scope		/
	
Apr�s
	
	rootScope	wrapper { clicked='VRAI' }
	^
	|
	scope		/
	

### Encapsuler les modifications de donn�es dans des services

Le second probl�me du point pr�c�dent peut �tre �vit� en n'utilisant pas de donn�es provenant de scopes parents ou du scope racine
mais en passant toujours par des services pour modifier les donn�es. 


### Garder des controlleurs simples

Essayez de garder des controlleurs petits et simples, avec peu de responsabilit�s. 

Un objectif est de par exemple r�-utiliser un controlleur avec plusieurs vues HTML diff�rentes : une vue mobile et une vue desktop sont un bon
cas d'�cole. Seul le code HTML/CSS va changer mais les controlleurs et services seront enti�rement r�utilis�s.

Une r�gle est de garder la logique de pr�sentation/navigation dans les controlleurs et la logique m�tier dans les services (communiquant 
avec d'�ventuels services distant au besoin).
	
	
### Passer dans le scope le plus vite possible

Afin de laisser angular executer le bindings, il faut que les �v�nements utilisateurs (susceptibles de modifier le r�sultat du binding) provoquent
un rafraichissement. C'est la finalit� de la m�thode ``$apply`` de scope qui executent un bloc de code puis r�-�value le binding.

	link: function(scope, elt, attrs) {
		elt.find('a').on('click', function(){
			scope.$apply(function(){
				// unfuck the event
			});
		});
	}

Cette m�thode est souvent appell�e en r�action � un �v�nement ext�rieur non encapsul� par angular (comme le font ng-click, $http, etc). 
Vu qu'angular est souvent tr�s m�content lors de l'imbrication des $apply, le plus simple est de le faire le plus vite possible. 
Cela afin d'avoir la libert� d'esprit et le confort de se dire que tout le code qu'on �crit se trouve dans le contexte d'�xecution du scope
sans se poser de questions ...

En dernier recours on peut r�cup�rer $rootScope.$$phase qui est vrai lorsque l'on est dans un $apply(), falsy sinon afin d'�viter une exception
si l'on appelle deux fois `$apply ` dans la m�me stack.


### Utiliser les filtres pour le formattage

Les filtres angular peuvent servir � toutes sortes de formattages, en �vitant de polluer le scope avec des fonction du genre :

	$scope.userLabel = function(user) {
		return user.firstName + ' ' + user.lastName;
	};
	
Ecrire plut�t un filtre :

	angular.module(...).filter('userLabel', function() {
		return function(user) {
			return user.firstName + ' ' + user.lastName;
		}
	});

Ce qui est tout de suite plus clair dans la vue:

	<span>bonjour {{user | userLabel }}</span>
	
L'avantage c'est que ce code de formattage est tout de suite mutualis� et injectable dans tous les autres composants du framework

	function MyService($filter) {
		var filterFn = $filter('userLabel');		// get the filter
		console.log(filterFn(user));
	}



### Router sans �tre d�rout�

Le routeur angular permet de structurer la navigation dans la page. Cela a plusieurs avantages :

* pour les utilisateurs d'avoir des URLs bookmarkables et une navigation native (bouton back)
* Pour les d�veloppeurs de cabler facilement de la navigation entre �crans d'une m�me application
* D'exposer facilement les diff�rentes fonctionnalit�s d'une application en regardant la table des routes, sous le m�me format
qu'une API REST


	/users		-> vue liste des utilisateurs
	/user/{id}	-> fiche d'un utilisateur
	/cars		-> vue liste des voitures
	/car/{id}	....

Dans le cas de navigation imbriqu�e (vue imbriqu�e dans une autre), il est conseill� d'utiliser le routeur externe 
[ui-router](https://github.com/angular-ui/ui-router) offrant plus de fonctionnalit�s.

Le mode HTML5 pur du routeur n�cessite une r�ecriture d'URL cot� serveur et est plus complexe � d�bugger. Il est conseill� de rester sur le
mode de compatibilit� utilisant des chemins apr�s le hashbang.

La modification de la table `search` du service $location permet de passer tout type de param�tre en plus des vues, attention car
le changement de search ne provoque pas de r�initialisation du controlleur :


	// controller of the view routed by /users/{id}, eg /users/42?mode=edit
	
	function UserController($routeParams){
		// get user id from route parameters. Each change of the route will 
		// create a new instance of the controllers
		var id = $routeParams.id;
		
		// search parameters are changed without reloading the controller, hence
		// the need to listen to events
		$route.$on('routeUpdate', function(){
			var mode = $location.search().mode;
		});
	}

### Comment communiquer avec un scope 'voisin'

Pour que deux scopes (ou autrement dit deux vues 'voisines' � l'�cran), communiquent ensemble, il n'est pas possible d'utiliser l'h�ritage du
scope car les vues n'ont pas de relations parent-enfant entre elles.

il existe alors trois moyens:

* Mettre dans les donn�es dans un scope parent commun, ou � d�faut le rootScope. Pas terrible comme dit pr�c�demment
* Utiliser un service commun. Le plus propre et le plus logique lorsque les deux vues partagent des donn�es. On voit alors l'inter�t de centraliser
des donn�es dans un service et d'utiliser une m�me r�f�rence qui sera partag�e. 
* Utiliser des ev�nements broadcast�s sur le $rootScope.  


### Utiliser les promises

Les services standard comme $http utilisent des promises pour g�rer facilement des encha�nements et imbrications d'appels asynchrones. 
Leur g�n�ralisation � des traitements qui ne serait que parfois asynchrones est aussi une bonne id�e. 

L'exemple parfait est la gestion d'un cache qui irait chercher des donn�es (retour asynchrone) ou pourrait les renvoyer imm�diatement (retour synchone).

	/**
	This function always return a promise, whenever the data is already in cache or not
	*/
	function getFromCache(key) {
		if (isInCache(key)) {
			var deferred = $q.defer();
			deferred.resolved(key);
			return deferred.promise;
		} else {
			return $http.get('/loadfromservice');
		}
	}
	
### Watch et unwatch

Une m�thode bien pratique est le `$watch` sur un scope qui permet d'enregistrer un binding comme le ferait une expression `{{expr}}` dans
un template. Cette m�thode cr�e un gestionnaire qui sera �valu� � chaque rafraichissement, d'o� la n�cessite de ne pas garder trop
d'expressions fantomes.

Dans le cas d'une directive, le $watch est li� au scope de la directive, donc dans le cas o� la directive poss�de son propre scope, il n'y
aura pas de probl�me de d�senregistement. N�anmoins si l'on ne veut que faire un watch temporaire, il faut conserver le r�sultat
de l'appel � `$watch()` qui est une fonction de nettoyage du watch.

	var deregisterFn = $scope.$watch('myExpr', function(){
		// do something
		
		// un register the watch immediately after its first call
		deregisterFn();
	});
	

### Utiliser les helpers angular (ou underscore)

Il existe quelques petites fonctions utilitaires � la racine du namespace angular : 

* `isString` `isArray` `isNumber` : trivialement nomm�es
* `copy` et `extend` pour cloner ou �tendre des objets 
*  les classiques `bind`, `forEach`

Certes moins puissants qu'une librairie comme underscore (qui est �galement fortement recommand� !)


## UI

### Penser au ng-cloak

L'ajout d'un attribut `ng-cloak` permet de cacher la visibilit� d'un template tant que le framework et l'application n'ont pas 
�t� compl�tement initialis�s. Cela permet d'�viter d'afficher des expressions `{{expr}}` non �valu�es dans les templates

### Utiliser des transclusions
TODO Pour cr�er des itemrenderers � la sauce flex.

## Performances et s�curit�

### binding = s�curit�

Les expressions de bindings dans les templates `{{expr}}` et `<div ng-bind="expr"></div>` utilisent une fonction d'�chappement des cha�nes
permettant d'�viter tout probl�me d'injection CSS ou JS dans la page par des donn�es malicieusement inject�es.  Cela fournit une protection
tr�s peu couteuse contre ce type d'injection sans avoir � recourir � une nettoyage en amont (requ�te ou stockage) plus complexe

### Mais �viter trop de bindings !!!

Toute la puissance du binding d'angular repose sur un m�canisme de dirty checking qui r�evalue toutes les expressions de bindings � chaque
�v�nement (utilisateur, timer, affichage, blur, r�seau). Ce m�canisme peut �tre couteux lorsque l'on commence � avoir trop d'expressions de bindings

- Limiter les constructions 'sans bornes', par exemple un tableau non-pagin�e avec beaucoup de colonnes.
- Utiliser le plugin batarang pour d�tecter les expressions trop couteuses, en particulier celle comportant des filtres 
- Pr�ferer des constructions simples. Dans le cas de classes CSS dynamique, on peut simplement placer une classe sur un �l�ment parent
et utiliser un s�lecteur pour retrouver des �l�ments fils plut�t que de r�p�ter X fois un `ng-class`  identique sur des �l�ments enfants

Il n'y a pas vraiment de limites maximum. Cela d�pendra de la complexit� des expressions, du type de VM javascript (chrome / IE8-), 
de la puissance du device (si mobile). Au del� de 1000-2000 bindings les effets se font ressentir. Cette limite est �galement un indicateur
d'user experience: 2000 bindings sur un �cran signifient 2000 informations dynamiques que le cerveau de l'utilisateur devront g�rer.




### M�moiser les �valuations de filtre

Certains filtres persos peuvent faire des traitement complexes pour formatter des donn�es. Comme les filtres sont r�-evalu�s � chaque
dirty-checking il est prudent de les optimiser
Si l'on sait que le r�sultat de l'�valuation sera toujours constant, on peut m�moriser le r�sultat dans l'objet source

	function userLabelFilter(user) {
		if (!user.__labelCache) {
			user.__labelCache = user.firstName + user.lastName;
		}
		return user.__labelCache;
		// instead of return user.firstName + user.lastName;
	}


### Bindonce

Lorsque les donn�es sont majoritairement statiques, la plupart des bindings n'ont pas besoin de s'executer � chaque rafraichissement.
La directive tierce [ bindonce](https://github.com/Pasvaz/bindonce) propose une alternative au binding standard particuli�rement
efficace dans le cas d'affichages dans des boucles avec ng-repeat.


### Pollings et timers

Le service $timeout a l'avantage de s'�xecuter dans le contexte du scope mais donc de r�evaluer tous les bindings. Si la fr�quence d'un
timer est trop �lev� (+ de 10/sec) il y a un risque de provoquer beaucoup de recalcul et donc un ralentissement de l'interface.

Il est alors conseiller de rebasculer sur une variante pur javascript `setTimeout` et de forcer le rafraichissement du scope que lorsqu'il
y a un changement utile. En plus la librairie underscore.js fournit une m�thode `throttle` permettant de limiter le d�bit d'appel d'une fonction


### Cache de templates
Les templates charg�s dynamiquement par les directives `ng-view`, `ng-include` ou dans la template d'une directive perso provoquent un
appel HTTP et donc une latence suppl�mentaire dans l'application.

Angular met en cache le r�sultat de tous ces chargement pendant toute la vie d'une application, mais en plus Il est possible de les pr�charger
par exemple dans la page principale de l'application. Il faut alors les inclure dans des blocs scripts avec des attributs :


	<script type="text/ng-template" id="/tpl.html">
		Content of the template.
	</script>
	
	<!-- won't load the template from the network since it is in cache -->
	<div ng-include="/tpl.html"></div>
	

### I18N cot� serveur

L'internationalisation des libell�s dans les templates en utilisant les bindings angular est � �viter si possible, pour limiter le nombre
de bindings dans l'application. 
Le co�t de performance induit par l'ajout de centaines de bindings suppl�mentaires par rapport � la possibilit� de changer de langue sans recharger
l'application est en g�n�ral d�favorable.

Les templates peuvent �tre traduits cot� serveurs, avec un fichier de traductions suppl�mentaire servies en JSON pour les libell�s 
g�n�r�s dynamiquement (dans les filtres angular par exemple). Les filtres standard d'angular (format de date, de monnaie etc) utilisent un fichier
de traduction disponible dans la distribution d'angular (angular-locale-fr_FR.js etc) qui ne supporte pas de changement dynamique.


## Mobile 

### Lancer manuellement

Pour acc�lerer le chargement de l'application dans un wrapper mobile (type bas� sur phonegap ou autre), il est conseill� de ne pas mettre le chargement
automatique par l'ajout de l'attribut `ng-app` sur un �l�ment du document. Il faut plut�t charger manuellement l'application 
avec `angular.bootstrap` lors de l'�v�nement d�di� lanc� par le wrapper.

Exemple pour cordova:

	document.addEventListener("deviceready", function(){
		angular.bootstrap(document.body, ["MyApp"]);
	});

### Laisser jqlite

Angular fonctionne avec ou sans jquery. Si jquery n'est pas n�cessaire, on peut donc s'en passer pour diminuer la taille des librairies.
Il faut n�anmoins faire attention de n'utiliser que les s�lecteurs support�s par jqlite lorsque l'on manipule le dom dans les directives.


### R�utiliser des controlleurs

Sans rentrer dans un d�bat sur le choix de d�veloppement d�di� / sp�cifique pour une application mobile, une approche relativement productive
est de conserver tout le code JS et de changer uniquement le code HTML /CSS (page principale, feuilles de styles, templates) pour faire
une version mobile. 
La conception d'angular permet d'avoir un couplage tr�s faible entre les vues et les mod�les/controlleurs car tout passe par le scope.

Si le fonctionnel des vues n'est pas trop divergent entre une application mobile et une application desktop, on peut ainsi r�utiliser une
tr�s grande proportion du code et de ne refaire que la pr�sentation des donn�es.

### Utiliser le module angular-mobile
Le module angular-mobile.js de la distribution r�gle le probl�me des �v�nements clicks trop long et ajout deux directives `ng-swipe-left`et
`ng-swipe-right` pour utiliser facilement des gestures de d�filement gauche/droite.


## Dev lifecycle

### Tester!
Un des points fort d'angular est sa philosophie tr�s orient�e TDD 

* L'injection de d�pendence facilite le test d'un composant unitairement en bouchonnant les d�pendences qui lui sont pass�s
* Le service `$http` standard supporte nativement de mocker toutes les requ�tes
* L'abstraction au maximum des manipulations du DOM permet d'executer un maximum de code dans une VM headless sans avoir de vrai navigateur
* Des outils facilitant le lancement de tests unitaires ou d'int�grations sont livr�s


### Compiler
La concat�nation et minification du code javascript est une �tape vitale pour la distribution d'une application javascript. Quelque soit
l'outil utilis�, il est important qu'il soit capable de g�rer le m�canisme d'injection de d�pendence par noms d'angular qui se retrouvera
cass� en cas de minification trop aggressive.

* grunt l'int�gre via une tache `ng-min` qui transforme les d�clarations implicites en explicites selon certains patterns
* google closure via une annotation de la JSDOC

Autant que possible il vaut mieux avoir un bon outil plut�t que de maintenir des tables d'injections � la main
	
	angular.service('MyService', ['$http', '$log', '$q', 'myService', function($http, $log, $q, myService, $timeout) {
		// SHIT HAPPENS because someone forgot to modify the dependency list
	}]);
	

### Batarang

[ batarang ](https://chrome.google.com/webstore/detail/angularjs-batarang/ighdmehidhipcmcojjgiloacoafjmpfk") est un plugin chrome pour
afficher la hi�rarchie des scopes et faire du profiling sur les expressions de binding les plus couteuses.

### Packager des librairies

En cr�ant des modules angular on tr�s facilement r�cuperer des librairies r�utilisables entre plusieurs projets. Un gestionnaire de package
javascript comme `bower` facilite grandement la tache de distribution des ces packages entre diff�rentes �quipes

A titre d'exemple, il y a plus de 450 librairies tierces pour angular packag�es dans le repository bower public !


 

