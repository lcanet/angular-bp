AngularJS Best practices
==========

## Structure et organisation du code

### Se fixer des conventions de nommages.
Pour tous les éléménts nommés, en JS, CSS et pour les identifieurs spécifiques à angular
Par exemple:

* Noms de services: pas de préfixe $ (réservé à angular), camelCase (ex: ``catalogService``) 
* Noms de classes CSS: tiret-case sans majuscules (``button-primary``)
* ID d'éléments: camelCase
* Noms d'évènements propagés sur les scopes : camelCase, en préfixant par un domaine (``Map.previewUpdated``)

### Séparer l'application en modules fonctionnels
Utiliser les modules angular pour regrouper un ensemble de composants (directives, controlleurs, filtrer, services) avec une logique
fonctionnelle. Ne pas utiliser de regroupement technique (1 module pour tous les filtres, 1 module pour tous les services etc).

Cette séparation technique peut (et devrait!) apparaitre en revanche dans l'arborescence de fichiers :

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


	
### Préfixer les filtres et directives

Afin d'éviter toute collisions d'éléments nommés dans les templates, il est prudent de préfixer toutes les directives et tous les filtres
spécifiques à un projet, avec un code propre à l'organisation ou au projet

	<pwet-button-panel label="{{user | pwetCamelize}}"></pwet-button-panel>
	

### Utiliser des modules

Utiliser des modules angular pour séparer l'application en bloc fonctionnels. Des modules peuvent également regrouper des éléments
communs pour former une librairie réutilisable dans plusieurs projets.  

	
## Utilisation des composants angular

### Ne Jamais référencer le DOM depuis les controllers
Il faut au maximum éviter la manipulation du DOM (objets javascript représentant des fragments de bouts de pages HTML en mémoire) dans les controllers
et les services angular.

Cela permet :

* de ne pas avoir de fuites mémoires (blocs de DOM dynamiques jamais référencés directement donc garbageable)
* de mieux découpler les différences responsabilités
* d'être facilement testable

Il faut toujours encapsuler les manipulations du DOM donc depuis les directives. Soit avec les directives fournies dans le framework
(show, ng-class, etc) soit avec des directives persos lorsque cela est nécessaire, par exemple lors de l'utilisation d'une librairie JS tierce. 
La méthode ``link`` d'une directive recoit l'élément de DOM (wrappé dans un objet jquery), que l'on peut ainsi manipuler à sa guise.


### Dans une directive, limiter la manipulation de DOM

Une directive manipulant du DOM doit essayer de se restreindre à l'élément qui lui est passé (perimètre d'application de la directive) ainsi qu'à ses
descendants. Lorsque l'on recherche des éléments descendants, toujours utiliser la méthode de recherche jquery ``find`` appliquée à l'élément 
racine de la directive passé à la méthode `` link`` et qui limitera ainsi les éléments à la sous arborescence.

	template: '<div><p class="raviolis"></p></div>',
    link: function(scope, elt, attrs) {
		// will only match raviolis in our directive instance and not all raviolis on screen
	    elt.find('.raviolis').show();
	}

	

### Penser au cycle de vie
Tout scope crée par angular peut également être détruit, par exemple si l'on détruit le bout de page associé. Les scopes détruits émettent un
évènement ``$destroy`` qui peut être écouté pour effectuer divers traitements :

* Nettoyage de données
* Desenregistrement de timers
* Dans le cas d'une directive, nettoyage d'évènements ou d'éléments ailleurs que sur le sous-arbre du DOM de la directive.

Un exemple classique de ce dernier cas est l'utilisation d'une directive encapsulant les composants bootstrap `` modal`` ou `` colorpicker``. 
Ces composants sont initialisés avec un bout de code javascript qui placent un morceau de DOM invisible à la fin du body.

Il faut alors capter l'évènement de destruction du scope pour appeler les bonnes méthodes de nettoyage.

	template: '<div></div>',
    link: function(scope, elt, attrs) {
	    var cp = elt.colorpicker();
		scope.$on('$destroy', function(){
			cp.picker.remove();		// remove the <div class="colorpicker"> at root of document
		});
	}

### Avoir des références de modèles immuables

Deux raisons 

1. avoir une donnée maitrisée par un service central et accessible en lecture dans les controlleurs et dans les scopes
2. modifier une donnée dans un scope parent

Pourquoi ? Dans le premier cas, si un service est responsable d'une donnée, il la présentera comme un membre du service et celle-ci
sera récupéré par le controlleur pour l'exposer dans le scope et la rendre disponible dans la vue :

	function MyController($scope, myService) {
		$scope.model = myService.model;
	}
	
Le code suivant ne sera executé qu'une seule fois à l'initialisation du controlleur. Si la référence est modifiée à un moment ultérieur 
(chargement asynchrone, ...), le scope aura une ancienne version du model qui ne sera plus bonne. Pas besoin de passer par une gestion
d'évènement pour cela : il suffit de renvoyer toujours une même référence immuable et le binding sur des attributs de modèle se chargera
du reste.

	<span>{{model.firstName}}</span>
	
Dans le service:
	
	service.populateModel = function() {
		service.model.firstName = '...';
		// et pas service.model = {firstName: '...'};
	};

Attention, le raccourci consistant à injecter directement le service dans le scope est franchement à bannir :

	function MyController($scope, myService) {
		$scope.service = myService;
	}
	
	<span>{{service.model.firstName}}</span>
	

Le second cas est une particularité de la gestion de l'héritage des scopes. Imaginons le code suivant:

	<span>{{clicked}}</span><a href="" ng-click="clicked = 'VRAI'">click me</a>
	
Pour évaluer le premier binding ``{{clicked}}``, la variable est d'abord évaluée dans le scope courant, puis si celle-ci n'est pas définie
dans le scope parent et ainsi de suite jusqu'au scope racine. 
Imaginons que dans notre cas la variable soit définie dans le scope racine. Le click, ne va pas s'appliquer à la variable {{clicked}} du scope
dans lequel elle est définie, mais directement dans le scope courant !

Avant:
	
	rootScope	clicked='FAUX'
	^
	|
	scope		/
	
Après
	
	rootScope	clicked='FAUX'
	^
	|
	scope		clicked='VRAI'
	
D'où un gros problème si d'autres scopes 'frangins' du scope courant utilisent également la variable ``clicked``.

La solution consiste à encapsuler les attributs modifiés dans un objet dont la référence sera immuable :

	<span>{{wrapper.clicked}}</span><a href="" ng-click="wrapper.clicked = 'VRAI'">click me</a>

Avant:
	
	rootScope	wrapper { clicked='FAUX' }
	^
	|
	scope		/
	
Après
	
	rootScope	wrapper { clicked='VRAI' }
	^
	|
	scope		/
	

### Encapsuler les modifications de données dans des services

Le second problème du point précédent peut être évité en n'utilisant pas de données provenant de scopes parents ou du scope racine
mais en passant toujours par des services pour modifier les données. 


### Garder des controlleurs simples

Essayez de garder des controlleurs petits et simples, avec peu de responsabilités. 

Un objectif est de par exemple ré-utiliser un controlleur avec plusieurs vues HTML différentes : une vue mobile et une vue desktop sont un bon
cas d'école. Seul le code HTML/CSS va changer mais les controlleurs et services seront entièrement réutilisés.

Une règle est de garder la logique de présentation/navigation dans les controlleurs et la logique métier dans les services (communiquant 
avec d'éventuels services distant au besoin).
	
	
### Passer dans le scope le plus vite possible

Afin de laisser angular executer le bindings, il faut que les évènements utilisateurs (susceptibles de modifier le résultat du binding) provoquent
un rafraichissement. C'est la finalité de la méthode ``$apply`` de scope qui executent un bloc de code puis ré-évalue le binding.

	link: function(scope, elt, attrs) {
		elt.find('a').on('click', function(){
			scope.$apply(function(){
				// unfuck the event
			});
		});
	}

Cette méthode est souvent appellée en réaction à un évènement extérieur non encapsulé par angular (comme le font ng-click, $http, etc). 
Vu qu'angular est souvent très mécontent lors de l'imbrication des $apply, le plus simple est de le faire le plus vite possible. 
Cela afin d'avoir la liberté d'esprit et le confort de se dire que tout le code qu'on écrit se trouve dans le contexte d'éxecution du scope
sans se poser de questions ...

En dernier recours on peut récupérer $rootScope.$$phase qui est vrai lorsque l'on est dans un $apply(), falsy sinon afin d'éviter une exception
si l'on appelle deux fois `$apply ` dans la même stack.


### Utiliser les filtres pour le formattage

Les filtres angular peuvent servir à toutes sortes de formattages, en évitant de polluer le scope avec des fonction du genre :

	$scope.userLabel = function(user) {
		return user.firstName + ' ' + user.lastName;
	};
	
Ecrire plutôt un filtre :

	angular.module(...).filter('userLabel', function() {
		return function(user) {
			return user.firstName + ' ' + user.lastName;
		}
	});

Ce qui est tout de suite plus clair dans la vue:

	<span>bonjour {{user | userLabel }}</span>
	
L'avantage c'est que ce code de formattage est tout de suite mutualisé et injectable dans tous les autres composants du framework

	function MyService($filter) {
		var filterFn = $filter('userLabel');		// get the filter
		console.log(filterFn(user));
	}



### Router sans être dérouté

Le routeur angular permet de structurer la navigation dans la page. Cela a plusieurs avantages :

* pour les utilisateurs d'avoir des URLs bookmarkables et une navigation native (bouton back)
* Pour les développeurs de cabler facilement de la navigation entre écrans d'une même application
* D'exposer facilement les différentes fonctionnalités d'une application en regardant la table des routes, sous le même format
qu'une API REST


	/users		-> vue liste des utilisateurs
	/user/{id}	-> fiche d'un utilisateur
	/cars		-> vue liste des voitures
	/car/{id}	....

Dans le cas de navigation imbriquée (vue imbriquée dans une autre), il est conseillé d'utiliser le routeur externe 
[ui-router](https://github.com/angular-ui/ui-router) offrant plus de fonctionnalités.

Le mode HTML5 pur du routeur nécessite une réecriture d'URL coté serveur et est plus complexe à débugger. Il est conseillé de rester sur le
mode de compatibilité utilisant des chemins après le hashbang.

La modification de la table `search` du service $location permet de passer tout type de paramètre en plus des vues, attention car
le changement de search ne provoque pas de réinitialisation du controlleur :


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

Pour que deux scopes (ou autrement dit deux vues 'voisines' à l'écran), communiquent ensemble, il n'est pas possible d'utiliser l'héritage du
scope car les vues n'ont pas de relations parent-enfant entre elles.

il existe alors trois moyens:

* Mettre dans les données dans un scope parent commun, ou à défaut le rootScope. Pas terrible comme dit précédemment
* Utiliser un service commun. Le plus propre et le plus logique lorsque les deux vues partagent des données. On voit alors l'interêt de centraliser
des données dans un service et d'utiliser une même référence qui sera partagée. 
* Utiliser des evènements broadcastés sur le $rootScope.  


### Utiliser les promises

Les services standard comme $http utilisent des promises pour gérer facilement des enchaînements et imbrications d'appels asynchrones. 
Leur généralisation à des traitements qui ne serait que parfois asynchrones est aussi une bonne idée. 

L'exemple parfait est la gestion d'un cache qui irait chercher des données (retour asynchrone) ou pourrait les renvoyer immédiatement (retour synchone).

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

Une méthode bien pratique est le `$watch` sur un scope qui permet d'enregistrer un binding comme le ferait une expression `{{expr}}` dans
un template. Cette méthode crée un gestionnaire qui sera évalué à chaque rafraichissement, d'où la nécessite de ne pas garder trop
d'expressions fantomes.

Dans le cas d'une directive, le $watch est lié au scope de la directive, donc dans le cas où la directive posséde son propre scope, il n'y
aura pas de problème de désenregistement. Néanmoins si l'on ne veut que faire un watch temporaire, il faut conserver le résultat
de l'appel à `$watch()` qui est une fonction de nettoyage du watch.

	var deregisterFn = $scope.$watch('myExpr', function(){
		// do something
		
		// un register the watch immediately after its first call
		deregisterFn();
	});
	

### Utiliser les helpers angular (ou underscore)

Il existe quelques petites fonctions utilitaires à la racine du namespace angular : 

* `isString` `isArray` `isNumber` : trivialement nommées
* `copy` et `extend` pour cloner ou étendre des objets 
*  les classiques `bind`, `forEach`

Certes moins puissants qu'une librairie comme underscore (qui est également fortement recommandé !)


## UI

### Penser au ng-cloak

L'ajout d'un attribut `ng-cloak` permet de cacher la visibilité d'un template tant que le framework et l'application n'ont pas 
été complètement initialisés. Cela permet d'éviter d'afficher des expressions `{{expr}}` non évaluées dans les templates

### Utiliser des transclusions
TODO Pour créer des itemrenderers à la sauce flex.

## Performances et sécurité

### binding = sécurité

Les expressions de bindings dans les templates `{{expr}}` et `<div ng-bind="expr"></div>` utilisent une fonction d'échappement des chaînes
permettant d'éviter tout problème d'injection CSS ou JS dans la page par des données malicieusement injectées.  Cela fournit une protection
très peu couteuse contre ce type d'injection sans avoir à recourir à une nettoyage en amont (requête ou stockage) plus complexe

### Mais éviter trop de bindings !!!

Toute la puissance du binding d'angular repose sur un mécanisme de dirty checking qui réevalue toutes les expressions de bindings à chaque
évènement (utilisateur, timer, affichage, blur, réseau). Ce mécanisme peut être couteux lorsque l'on commence à avoir trop d'expressions de bindings

- Limiter les constructions 'sans bornes', par exemple un tableau non-paginée avec beaucoup de colonnes.
- Utiliser le plugin batarang pour détecter les expressions trop couteuses, en particulier celle comportant des filtres 
- Préferer des constructions simples. Dans le cas de classes CSS dynamique, on peut simplement placer une classe sur un élément parent
et utiliser un sélecteur pour retrouver des éléments fils plutôt que de répéter X fois un `ng-class`  identique sur des éléments enfants

Il n'y a pas vraiment de limites maximum. Cela dépendra de la complexité des expressions, du type de VM javascript (chrome / IE8-), 
de la puissance du device (si mobile). Au delà de 1000-2000 bindings les effets se font ressentir. Cette limite est également un indicateur
d'user experience: 2000 bindings sur un écran signifient 2000 informations dynamiques que le cerveau de l'utilisateur devront gérer.




### Mémoiser les évaluations de filtre

Certains filtres persos peuvent faire des traitement complexes pour formatter des données. Comme les filtres sont ré-evalués à chaque
dirty-checking il est prudent de les optimiser
Si l'on sait que le résultat de l'évaluation sera toujours constant, on peut mémoriser le résultat dans l'objet source

	function userLabelFilter(user) {
		if (!user.__labelCache) {
			user.__labelCache = user.firstName + user.lastName;
		}
		return user.__labelCache;
		// instead of return user.firstName + user.lastName;
	}


### Bindonce

Lorsque les données sont majoritairement statiques, la plupart des bindings n'ont pas besoin de s'executer à chaque rafraichissement.
La directive tierce [ bindonce](https://github.com/Pasvaz/bindonce) propose une alternative au binding standard particulièrement
efficace dans le cas d'affichages dans des boucles avec ng-repeat.


### Pollings et timers

Le service $timeout a l'avantage de s'éxecuter dans le contexte du scope mais donc de réevaluer tous les bindings. Si la fréquence d'un
timer est trop élevé (+ de 10/sec) il y a un risque de provoquer beaucoup de recalcul et donc un ralentissement de l'interface.

Il est alors conseiller de rebasculer sur une variante pur javascript `setTimeout` et de forcer le rafraichissement du scope que lorsqu'il
y a un changement utile. En plus la librairie underscore.js fournit une méthode `throttle` permettant de limiter le débit d'appel d'une fonction


### Cache de templates
Les templates chargés dynamiquement par les directives `ng-view`, `ng-include` ou dans la template d'une directive perso provoquent un
appel HTTP et donc une latence supplémentaire dans l'application.

Angular met en cache le résultat de tous ces chargement pendant toute la vie d'une application, mais en plus Il est possible de les précharger
par exemple dans la page principale de l'application. Il faut alors les inclure dans des blocs scripts avec des attributs :


	<script type="text/ng-template" id="/tpl.html">
		Content of the template.
	</script>
	
	<!-- won't load the template from the network since it is in cache -->
	<div ng-include="/tpl.html"></div>
	

### I18N coté serveur

L'internationalisation des libellés dans les templates en utilisant les bindings angular est à éviter si possible, pour limiter le nombre
de bindings dans l'application. 
Le coût de performance induit par l'ajout de centaines de bindings supplémentaires par rapport à la possibilité de changer de langue sans recharger
l'application est en général défavorable.

Les templates peuvent être traduits coté serveurs, avec un fichier de traductions supplémentaire servies en JSON pour les libellés 
générés dynamiquement (dans les filtres angular par exemple). Les filtres standard d'angular (format de date, de monnaie etc) utilisent un fichier
de traduction disponible dans la distribution d'angular (angular-locale-fr_FR.js etc) qui ne supporte pas de changement dynamique.


## Mobile 

### Lancer manuellement

Pour accélerer le chargement de l'application dans un wrapper mobile (type basé sur phonegap ou autre), il est conseillé de ne pas mettre le chargement
automatique par l'ajout de l'attribut `ng-app` sur un élément du document. Il faut plutôt charger manuellement l'application 
avec `angular.bootstrap` lors de l'évènement dédié lancé par le wrapper.

Exemple pour cordova:

	document.addEventListener("deviceready", function(){
		angular.bootstrap(document.body, ["MyApp"]);
	});

### Laisser jqlite

Angular fonctionne avec ou sans jquery. Si jquery n'est pas nécessaire, on peut donc s'en passer pour diminuer la taille des librairies.
Il faut néanmoins faire attention de n'utiliser que les sélecteurs supportés par jqlite lorsque l'on manipule le dom dans les directives.


### Réutiliser des controlleurs

Sans rentrer dans un débat sur le choix de développement dédié / spécifique pour une application mobile, une approche relativement productive
est de conserver tout le code JS et de changer uniquement le code HTML /CSS (page principale, feuilles de styles, templates) pour faire
une version mobile. 
La conception d'angular permet d'avoir un couplage très faible entre les vues et les modèles/controlleurs car tout passe par le scope.

Si le fonctionnel des vues n'est pas trop divergent entre une application mobile et une application desktop, on peut ainsi réutiliser une
très grande proportion du code et de ne refaire que la présentation des données.

### Utiliser le module angular-mobile
Le module angular-mobile.js de la distribution régle le problème des événements clicks trop long et ajout deux directives `ng-swipe-left`et
`ng-swipe-right` pour utiliser facilement des gestures de défilement gauche/droite.


## Dev lifecycle

### Tester!
Un des points fort d'angular est sa philosophie très orientée TDD 

* L'injection de dépendence facilite le test d'un composant unitairement en bouchonnant les dépendences qui lui sont passés
* Le service `$http` standard supporte nativement de mocker toutes les requêtes
* L'abstraction au maximum des manipulations du DOM permet d'executer un maximum de code dans une VM headless sans avoir de vrai navigateur
* Des outils facilitant le lancement de tests unitaires ou d'intégrations sont livrés


### Compiler
La concaténation et minification du code javascript est une étape vitale pour la distribution d'une application javascript. Quelque soit
l'outil utilisé, il est important qu'il soit capable de gérer le mécanisme d'injection de dépendence par noms d'angular qui se retrouvera
cassé en cas de minification trop aggressive.

* grunt l'intégre via une tache `ng-min` qui transforme les déclarations implicites en explicites selon certains patterns
* google closure via une annotation de la JSDOC

Autant que possible il vaut mieux avoir un bon outil plutôt que de maintenir des tables d'injections à la main
	
	angular.service('MyService', ['$http', '$log', '$q', 'myService', function($http, $log, $q, myService, $timeout) {
		// SHIT HAPPENS because someone forgot to modify the dependency list
	}]);
	

### Batarang

[ batarang ](https://chrome.google.com/webstore/detail/angularjs-batarang/ighdmehidhipcmcojjgiloacoafjmpfk") est un plugin chrome pour
afficher la hiérarchie des scopes et faire du profiling sur les expressions de binding les plus couteuses.

### Packager des librairies

En créant des modules angular on très facilement récuperer des librairies réutilisables entre plusieurs projets. Un gestionnaire de package
javascript comme `bower` facilite grandement la tache de distribution des ces packages entre différentes équipes

A titre d'exemple, il y a plus de 450 librairies tierces pour angular packagées dans le repository bower public !


 

