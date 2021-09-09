# Exemples de requête DAX construite pour des besoins précis

## Créer un table synthétique de mouvement de crédit et débit en fonction d'une liste SharePoint

Dans ce cas de figure j'ai une liste SharePoint qui me permet de saisir un mouvement de budget d'une activité vers une autre. Le problème c'est que je ne peux avoir que la somme des crédit mais pas la somme négative des débits.

Il fallait donc que je construise une table intermédiaire pour obtenir pour chaque activité la somme des crédits et la somme des débits.

Pour cela j'utilise les fonctions DAX 'UNION' et 'SUMMARIZE' en regroupant la somme de toutes les activités par Id de destination (Crédit) et la somme mutiplié par -1 de toutes les activités par Id d'origine (Crédit).

    Débit et crédit des mouvements par = UNION(
        SUMMARIZE(
            'Mouvements',
            'Mouvements'[ActiviteDestinationId],
            "Montants",
            SUM('Mouvements'[Montant])
            ),
        SUMMARIZE(
            'Mouvements',
            'Mouvements'[ActiviteOrigineId],
            "Montants",
            SUM('Mouvements'[Montant])*-1)
            )

## Obtenir une note sous forme d'étoile

Un de mes clients souhaitait avoir une note pour afficher la qualité du budget en cours sur tableau de bord de suivi budgétaire.

Pour cela il me fallait d'un côté pouvoir déterminer les notes en fonction des critères choisis et d'autre part afficher des étoiles.

Pour la détermination des notes j'ai utiliser une première mesure qui donne des points (0 ou 0,5 ou 1) en fonction de la valeurs de 4 mesure différentes affiché en pourcentage. 

Par défaut si la valeur est vide on donne 1 point et de même si la valeur est dessous de 1 (100%) sinon on donne 0,5 entre 1 et 1,05 (100%-105%) et 0 au dessus de 1,05 (105%) :

    Note de la synthèse budgétaire = 
    If(NOT ISBLANK(Mesures[% Planifié]),
            If(Mesures[% Planifié]<=1,1,0) + 
            If(Mesures[% Planifié]>=1 && Mesures[% Planifié]<1.05,0.5,0)
        ,1)
    +
    If(NOT ISBLANK(Mesures[% Commandé]),
        If(Mesures[% Commandé]<=1,1,0) + 
        If(Mesures[% Commandé]>=1 && Mesures[% Commandé]<1.05,0.5,0)
        ,1)
    +
    If(NOT ISBLANK(Mesures[% Engagé]),
        If(Mesures[% Engagé]<=1,1,0) + 
        If(Mesures[% Engagé]>=1 && Mesures[% Engagé]<1.05,0.5,0)
        ,1)
    +
    If(NOT ISBLANK(Mesures[% Consommé]),
        If(Mesures[% Consommé]<=1,1,0) + 
        If(Mesures[% Consommé]>=1 && Mesures[% Consommé]<1.05,0.5,0)
        1)

Pour l'affichage sous forme d'étoile le code est maintenant directemment inclus par Power BI via son assistant mais on peut si nécessaire modifier les variables pour son besoin.

Dans notre cas la note ne peut pas dépassé 4 et et nous avons des demis points soit 8 valeurs possibles, les étoiles ne pouvant être rempli à moitié il nous en faut donc 8.

Ce qui donne la mesure suivante :

    Note étoilé de la synthèse budgétaire = 
    VAR __MAX_NUMBER_OF_STARS = 8
    VAR __MIN_RATED_VALUE = 0
    VAR __MAX_RATED_VALUE = 4
    VAR __BASE_VALUE = [Note de la synthèse budgétaire]
    VAR __NORMALIZED_BASE_VALUE =
        MIN(
            MAX(
                DIVIDE(
                    __BASE_VALUE - __MIN_RATED_VALUE,
                    __MAX_RATED_VALUE - __MIN_RATED_VALUE
                ),
                0
            ),
            1
        )
    VAR __STAR_RATING = ROUND(__NORMALIZED_BASE_VALUE * __MAX_NUMBER_OF_STARS, 0)
    RETURN
        IF(
            NOT ISBLANK(__BASE_VALUE),
            REPT(UNICHAR(9733), __STAR_RATING)
                & REPT(UNICHAR(9734), __MAX_NUMBER_OF_STARS - __STAR_RATING)
        )


## Calculer un montant des commande cumulés qui ne sera pas impacté par les filtres

Pour cela on va utiliser la fonction 'ALL' qui dans notre cas sera mixer avec un 'Filter' pour obtenir toutes les commandes dont la date de signature est inférieur ou égale à la valeur 'MAX' et toutes les commandes ayant le statut 'Signé' via la fonction 'IN'

    Commandes cumulées = CALCULATE(
        SUM(Commandes[Montant total TTC]),
        FILTER(
            ALL(Commandes[Date signature],'Commandes'[Statut technique]),
            Commandes[Date signature]<=MAX(Commandes[Date signature]) &&
            'Commandes'[Statut technique] IN { "Signé" })
        )

Il est à noter que le fonction 'IN' pourrait aussi nous permettre de préciser plusieurs statuts différents en les séparant par une virgule. 