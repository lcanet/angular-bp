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
Utiliser les modules fonctionnels pour regrouper un ensemble de composants (directives, controlleurs, filtrer, services) avec une logique
fonctionnelle. Ne pas utiliser de regroupement technique (1 module pour tous les filtres, 1 module pour tous les services).

Cette s�paration technique peut apparaitre en revanche dans l'arborescence de fichiers :

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


	
## Utilisation des composants angular

### Ne Jamais r�f�rencer le DOM depuis les controllers
Il faut au maximum �viter la manipulation du DOM (objets javascript repr�sentant des fragments de bouts de pages HTML en m�moire) dans les controllers
et les services angular.

Cela permet :

* de ne pas avoir de fuites m�moires (blocs de DOM dynamiques pas r�f�renc�s directement donc garbageable)
* de mieux d�coupler les diff�rences responsabilit�s
* d'�tre facilement plus testable

Il faut toujours encapsuler les manipulations du DOM donc depuis les directives. Soit avec les directives classiques (show, ng-class, etc) soit
avec des directives persos lorsque cela est n�cessaire, par exemple lors de l'utilisation d'une librairie JS tierce. 
La m�thode ``link`` d'une directive recoit l'�l�ment de DOM (wrapp� dans un objet jquery), que l'on peut ainsi manipuler � sa guise.

### Dans une directive, limiter la manipulation de DOM
Une directive manipulant du DOM doit essayer de se restreindre � l'�l�ment qui lui est pass� (perim�tre d'application de la directive) ainsi qu'� ses
descendants. Lorsque l'on recherche des �l�ments descendants, toujours utiliser la m�thode de recherche jquery ``find`` appliqu�e � l'�l�ment 
racine de la directive pass� � la m�thode `` link`` et qui limitera ainsi les �l�ments � la sous arborescence.

	template: '<div><p class="raviolis"></p></div>',
    link: function(scope, elt, attrs) {
		// will only match raviolis in our directive instance
	    elt.find('.raviolis').show();
	}

	

### Penser au cycle de vie
Tout scope cr�e par angular peut �galement �tre d�truit, par exemple si l'on d�truit le bout de page associ�. Les scopes d�truits �mettent un
�v�nement ``$destroy`` qui peut �tre �cout� pour effectuer divers traitements :

* Nettoyage de donn�es
* Desenregistrement de timers
* Dans le cas d'une directive, nettoyage d'�v�nements ou d'�l�ments ailleurs que sur le sous-arbre du DOM de la directive.

Un exemple classique est l'utilisation d'une directive encapsulant les composants bootstrap `` modal`` ou `` colorpicker``. Ces composants
sont initialis�s avec un bout de code javascript qui placent un morceau de DOM invisible � la fin du body (typiquement pour g�rer une popin).
Il faut alors capter l'�v�nement de destruction du scope pour appeler les bonnes m�thodes de nettoyage.

	template: '<div></div>',
    link: function(scope, elt, attrs) {
	    var cp = elt.colorpicker();
		scope.$on('$destroy', function(){
			cp.picker.remove();		// remove the <div class="colorpicker"> at root of document
		});
	}

### Garder des controlleurs simples

Essayez de garder des controlleurs petits et simples (voir aussi le tip suivant), avec peu de responsabilit�s. 

Un objectif est de par exemple r�-utiliser un controlleur avec plusieurs vues HTML diff�rentes : une vue mobile et vue desktop sont un bon
cas d'�cole. 

### Encapsuler les modifications de donn�es dans des services

Une r�gle est de garder la logique de pr�sentation/navigation dans les controlleurs et la logique m�tier dans les services (communiquant 
avec d'�ventuels services distant au besoin).


### Avoir des r�f�rences de mod�les immuables

Deux raisons 

1. avoir une donn�e maitris�e par un service central et accessible en lecture dans les controlleurs et dans les scopes
2. modifier une donn�e dans un scope parent

Pourquoi ? Dans le premier cas, si un service est responsable d'une donn�e, il la pr�sentera comme un membre du service et celle-ci
sera r�cup�r� par le controlleur.

	function MyController($scope, myService) {
		$scope.model = myService.model;
	}
	
Le code suivant ne sera execut� qu'une seule fois � l'initialisation du service. Si la r�f�rence est modifi�e � un moment ult�rieur 
(chargement asynchrone, ...), le scope aura une ancienne version du model qui ne sera plus bonne. Pas besoin de passer par une gestion
d'�v�nement pour cela : il suffit de renvoyer toujours une m�me r�f�rence immuable et le binding sur des attributs de mod�le se chargera
du reste.

	<span>{{model.firstName}}</span>
	
	Dans le service:
	
	service.populateModel = function() {
		service.model.firstName = '...';
		// et pas service.model = {firstName: '...'};
	};

Attention, le raccourci consistant � injecter directement le modele dans le scope est franchement � bannir :

	function MyController($scope, myService) {
		$scope.service = myService;
	}
	
	<span>{{service.model.firstName}}</span>
	

Le second cas est une particularit� de la gestion de l'h�ritage des scopes. Imaginons les code suivantes:

	<span>{{clicked}}</span><a href="" ng-click="clicked = 'VRAI'">click me</a>
	
Pour �valuer le premier binding ``{{clicked}}``, la variable est d'abord �valu�e dans le scope courant, puis si celle-ci n'est pas d�finie
dans le scope parent et ainsi de suite jusqu'au scope racine. 
Imaginons que dans notre cas la variable soit d�finie dans le scope racine. Le click, ne va pas s'appliquer � la variable {{clicked}} du scope
dans lequel elle est d�finie, mais directmenet dans le scope courant !

Avant:
	
	rootScope	clicked='FAUX'
	|
	scope		/
	
Apr�s
	
	rootScope	clicked='FAUX'
	|
	scope		clicked='VRAI'
	
D'o� un gros probl�me si d'autres scopes 'frangins' du scope courant utilisent �galement la variable ``clicked``.

La solution consiste � encapsuler les attributs modifi�s dans un object dont la r�f�rence sera immuable :

	<span>{{wrapper.clicked}}</span><a href="" ng-click="wrapper.clicked = 'VRAI'">click me</a>

Avant:
	
	rootScope	wrapper { clicked='FAUX' }
	|
	scope		/
	
Apr�s
	
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
Pour cr�er des itemrenderers � la sauce flex.

## Performances et s�curit�

### binding = s�curit�

### Mais �viter trop de bindings !!!
Les meilleures pratiques consistent donc � utiliser au maximum le binding d'angular

### Batarang

### M�moiser les �valuations de filtre

### Bindonce

### Pollings et timers

### Cache de templates


## Tests





 

