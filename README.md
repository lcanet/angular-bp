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
Utiliser les modules fonctionnels pour regrouper un ensemble de composants (directives, controlleurs, filtrer, services) avec une logique
fonctionnelle. Ne pas utiliser de regroupement technique (1 module pour tous les filtres, 1 module pour tous les services).

Cette séparation technique peut apparaitre en revanche dans l'arborescence de fichiers :

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


	
## Utilisation des composants angular

### Ne Jamais référencer le DOM depuis les controllers
Il faut au maximum éviter la manipulation du DOM (objets javascript représentant des fragments de bouts de pages HTML en mémoire) dans les controllers
et les services angular.

Cela permet :

* de ne pas avoir de fuites mémoires (blocs de DOM dynamiques pas référencés directement donc garbageable)
* de mieux découpler les différences responsabilités
* d'être facilement plus testable

Il faut toujours encapsuler les manipulations du DOM donc depuis les directives. Soit avec les directives classiques (show, ng-class, etc) soit
avec des directives persos lorsque cela est nécessaire, par exemple lors de l'utilisation d'une librairie JS tierce. 
La méthode ``link`` d'une directive recoit l'élément de DOM (wrappé dans un objet jquery), que l'on peut ainsi manipuler à sa guise.

### Dans une directive, limiter la manipulation de DOM
Une directive manipulant du DOM doit essayer de se restreindre à l'élément qui lui est passé (perimètre d'application de la directive) ainsi qu'à ses
descendants. Lorsque l'on recherche des éléments descendants, toujours utiliser la méthode de recherche jquery ``find`` appliquée à l'élément 
racine de la directive passé à la méthode `` link`` et qui limitera ainsi les éléments à la sous arborescence.

	template: '<div><p class="raviolis"></p></div>',
    link: function(scope, elt, attrs) {
		// will only match raviolis in our directive instance
	    elt.find('.raviolis').show();
	}

	

### Penser au cycle de vie
Tout scope crée par angular peut également être détruit, par exemple si l'on détruit le bout de page associé. Les scopes détruits émettent un
évènement ``$destroy`` qui peut être écouté pour effectuer divers traitements :

* Nettoyage de données
* Desenregistrement de timers
* Dans le cas d'une directive, nettoyage d'évènements ou d'éléments ailleurs que sur le sous-arbre du DOM de la directive.

Un exemple classique est l'utilisation d'une directive encapsulant les composants bootstrap `` modal`` ou `` colorpicker``. Ces composants
sont initialisés avec un bout de code javascript qui placent un morceau de DOM invisible à la fin du body (typiquement pour gérer une popin).
Il faut alors capter l'évènement de destruction du scope pour appeler les bonnes méthodes de nettoyage.

	template: '<div></div>',
    link: function(scope, elt, attrs) {
	    var cp = elt.colorpicker();
		scope.$on('$destroy', function(){
			cp.picker.remove();		// remove the <div class="colorpicker"> at root of document
		});
	}

### Garder des controlleurs simples

Essayez de garder des controlleurs petits et simples (voir aussi le tip suivant), avec peu de responsabilités. 

Un objectif est de par exemple ré-utiliser un controlleur avec plusieurs vues HTML différentes : une vue mobile et vue desktop sont un bon
cas d'école. 

### Encapsuler les modifications de données dans des services

Une règle est de garder la logique de présentation/navigation dans les controlleurs et la logique métier dans les services (communiquant 
avec d'éventuels services distant au besoin).


### Avoir des références de modèles immuables

Deux raisons 

1. avoir une donnée maitrisée par un service central et accessible en lecture dans les controlleurs et dans les scopes
2. modifier une donnée dans un scope parent

Pourquoi ? Dans le premier cas, si un service est responsable d'une donnée, il la présentera comme un membre du service et celle-ci
sera récupéré par le controlleur.

	function MyController($scope, myService) {
		$scope.model = myService.model;
	}
	
Le code suivant ne sera executé qu'une seule fois à l'initialisation du service. Si la référence est modifiée à un moment ultérieur 
(chargement asynchrone, ...), le scope aura une ancienne version du model qui ne sera plus bonne. Pas besoin de passer par une gestion
d'évènement pour cela : il suffit de renvoyer toujours une même référence immuable et le binding sur des attributs de modèle se chargera
du reste.

	<span>{{model.firstName}}</span>
	
	Dans le service:
	
	service.populateModel = function() {
		service.model.firstName = '...';
		// et pas service.model = {firstName: '...'};
	};

Attention, le raccourci consistant à injecter directement le modele dans le scope est franchement à bannir :

	function MyController($scope, myService) {
		$scope.service = myService;
	}
	
	<span>{{service.model.firstName}}</span>
	

Le second cas est une particularité de la gestion de l'héritage des scopes. Imaginons les code suivantes:

	<span>{{clicked}}</span><a href="" ng-click="clicked = 'VRAI'">click me</a>
	
Pour évaluer le premier binding ``{{clicked}}``, la variable est d'abord évaluée dans le scope courant, puis si celle-ci n'est pas définie
dans le scope parent et ainsi de suite jusqu'au scope racine. 
Imaginons que dans notre cas la variable soit définie dans le scope racine. Le click, ne va pas s'appliquer à la variable {{clicked}} du scope
dans lequel elle est définie, mais directmenet dans le scope courant !

Avant:
	
	rootScope	clicked='FAUX'
	|
	scope		/
	
Après
	
	rootScope	clicked='FAUX'
	|
	scope		clicked='VRAI'
	
D'où un gros problème si d'autres scopes 'frangins' du scope courant utilisent également la variable ``clicked``.

La solution consiste à encapsuler les attributs modifiés dans un object dont la référence sera immuable :

	<span>{{wrapper.clicked}}</span><a href="" ng-click="wrapper.clicked = 'VRAI'">click me</a>

Avant:
	
	rootScope	wrapper { clicked='FAUX' }
	|
	scope		/
	
Après
	
	rootScope	wrapper { clicked='VRAI' }
	|
	scope		/
	



	
	
### Passer dans le scope le plus vite possible

### Utiliser les filtres pour le formattage


### Comment communiquer avec un scope 'voisin'
Events ou service

### Utiliser les promises
exemple du cache

### Utiliser les primitives angular
.copy, .extend, isString etc


## UI

### Penser au ng-cloak

### Utiliser des transclusions
Pour créer des itemrenderers à la sauce flex.

## Performances et sécurité

### binding = sécurité

### Mais éviter trop de bindings !!!
Les meilleures pratiques consistent donc à utiliser au maximum le binding d'angular

### Batarang

### Mémoiser les évaluations de filtre

### Bindonce

### Pollings et timers

### Cache de templates


## Tests





 

