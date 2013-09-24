AngularJS Best practices
==========

## Structure et organisation du code

### Se fixer des conventions de nommages.
Pour tous les éléménts nommés, en JS, CSS et pour les identifieurs spécifiques à angular
Par exemple:

* Noms de services: pas de préfixe $ (réservé à angular), camelCase (ex: ``catalogService``)
* Noms de classes CSS: tiret-case sans majuscules (``button-primary``)
* ID d'éléments: camelCase

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




### Encapsuler les modifications de données dans des services

### Avoir des références de modèles immuables

### Préfixer les filtres et directives

### Passer dans le scope le plus vite possible

### Garder des controlleurs simples

### Utiliser les filtres pour le formattage

### Comment communiquer avec un scope 'voisin'
Events ou service

### Utiliser les promises
exemple du cache

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





 

