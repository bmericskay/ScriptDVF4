---
title: "Script 4"
author: "Boris Mericskay et Florent Demoraes"
date: "12/11/2021"
output: html_document


---
LES DONNÉES DVF À UNE ÉCHELLE LOCALE : LES EXEMPLES DE LA MÉTROPOLE RÉGIONALE ET DE LA COMMUNE DE RENNES
---

Ce script présente toutes les étapes de manipulation de la dimension spatiale des données DVF et propose plusieurs formes de représentation cartographique de ces données à une échelle métropolitaine (Rennes Métropole).


## Préparation du projet

### Définition de l'environnement de travail

On définit ici le dossier qui centralise les données et où les différents jeux de données seront exportés

```{r setup, include=FALSE} 
knitr::opts_knit$set(root.dir = 'C:/DVF')
knitr::opts_chunk$set(warning = FALSE, message = FALSE) 

```
### Chargement des packages R nécessaires


```{r}
library(tidyverse) #Manipulation de données
library(sf) #Manipumation de données spatiales
library(cartography) #Cartographie thématique
```

### Import du jeu de données brut (si nécessaire)
```{r}
DVF <- read.csv("DATA/DVF_brut.csv", encoding="UTF-8", stringsAsFactors=FALSE)
```

---
## 1-Créer le jeu de données correspondant à Rennes métropole (RM)


### Extraire les mutations de Rennes Métropole
```{r}
MutationsRM <- MutationsBZH %>% filter(EPCI == 'Rennes Métropole')
```

### Tableau récapitulatif des indicateurs immobiliers de RM

```{r}
RecapRM <- MutationsRM %>% group_by(type) %>% 
  summarise(tot = n (), prixmed = median(prix), prixmoy = mean(prix), surfmed = median(surface), surfmoy = mean(surface), prixm2med = median(prixm2), prixm2moy = mean(prixm2))

RecapRM <- RecapRM %>% mutate(part = (tot/sum(tot)*100))

print(RecapRM)
```


---
## 2-Exploration des prix et des volumes à l'échelon des IRIS


### Importer la couche des IRIS (IGN)
```{r}
IRISRM <- st_read(dsn = "DATA/IRISRM.shp", stringsAsFactors = FALSE)
plot(IRISRM["ccom"])
```

### Calculer et cartographier le prix moyen au m² par IRIS

```{r}
# Réimporter si nécessaire la couche des communes de Bretagne (Admin Express IGN) et la reprojeter
Communes <- st_read(dsn = "DATA/Communes.shp", stringsAsFactors = FALSE)

# Calcul du prix moyen au m² par IRIS
IRISDVF <- IRISRM %>% st_join(MutationsRM) %>% group_by(code_iris) %>% summarise(Prixm2 = mean(prixm2))

# Sélection des communes dont le nom sera affiché sur la carte
liste_noms_RM <- c("Rennes", "Cesson-Sévigné", "Chartres-de-Bretagne", "Bécherel", "Mordelles", "Corps-Nuds", "Chevaigné")
Selection_Communes_RM <- Communes[which(Communes$NOM_COM %in% liste_noms_RM),]

# Paramétrage des marges de la fenêtre pour maximiser l'emprise de la carte
par(mar = c(0, 0, 1.2, 3))

# Affichage des communes de RM et alentours en arrière-plan et centrage de la carte sur Rennes Métropole
plot(st_geometry(Communes), lwd = 0.1, col = "grey95", border = "white", bg = "#B5D0D0", xlim = c(st_bbox(IRISRM)[1], st_bbox(IRISRM)[3]), ylim = c(st_bbox(IRISRM)[2], st_bbox(IRISRM)[4])) 

choroLayer(
  x = IRISDVF,
  var = "Prixm2",
  breaks = c(1250, 1500, 2000, 2500, 3000, 3600),
  col = c("#1a9641", "#a6d96a", "#ffffbf", "#fdae61", "#d7191c"),
  add = TRUE,
  border = "black",
  legend.nodata = "Aucune mutation",
  legend.border = "white",
  legend.title.txt = "Prix moyen \nau m² (Euros)")

layoutLayer(title = "Prix moyen au m² de l'immobilier par IRIS dans Rennes Métropole (2014-2019)", source = "         IGN et DGFip, 2021", 
            north = TRUE, horiz = FALSE, tabtitle = TRUE,
            frame = FALSE, col = "#cdd2d4", coltitle = "#8A5543")

# Ajouter les étiquettes des communes selectionnées plus haut
labelLayer(Selection_Communes_RM, txt = "NOM_COM", halo = TRUE, bg = "white", r = 0.05, cex = 0.7, pos = 3, font = 3, offset = 0, overlap = TRUE)

```

### Calculer et cartographier le nombre de mutations par IRIS
```{r}
# Calcul du nombre de mutations par IRIS
IRISDVF <- IRISDVF %>% mutate(Nbmutations = lengths(st_intersects(IRISDVF, MutationsRM)))
IRISDVF <- IRISDVF %>% na.omit() # pour supprimer les valeurs manquantes

# Paramétrage des marges de la fenêtre pour maximiser l'emprise de la carte
par(mar = c(0, 0, 1.2, 3))

# Affichage des communes de RM et alentours en arrière-plan et centrage de la carte sur Rennes Métropole
plot(st_geometry(Communes), lwd = 0.1, col = "grey95", border = "white", bg = "#B5D0D0", xlim = c(st_bbox(IRISRM)[1], st_bbox(IRISRM)[3]), ylim = c(st_bbox(IRISRM)[2], st_bbox(IRISRM)[4])) 

# Affichage des IRIS
plot(st_geometry(IRISRM), # appel du jeu de données
     border = "grey80", # couleur de la bordure des IRIS
     add = TRUE,
     lwd = 0.05) 

# Affichage des symboles proportionnels (nombre de mutations)
propSymbolsLayer(x = IRISDVF, # appel du jeu de données
                 var = "Nbmutations", # appel de la variable à cartographier
                 col = "#2ECC40", # couleur cercles
                 border = "white",  # couleur bordure cercle
                 lwd = 0.01,
                 inches = 0.08, # Taille des cercles
                 add = TRUE,
                 legend.pos = "n",
                 ) 

layoutLayer(title = "Nombre de mutations par IRIS dans Rennes Métropole (2014-2019)", source = "         IGN et DGFip, 2021", north = TRUE, horiz = FALSE, tabtitle = TRUE, frame = FALSE, col = "#cdd2d4", coltitle = "#8A5543")

legendCirclesSymbols(
  pos = "bottomleft",
  title.txt = "Nombre de mutations DVF",
  title.cex = 0.8,
  cex = 1,
  border = "white",
  lwd = 0.01,
  values.cex = 0.6,
  var = c(min(IRISDVF$Nbmutations), max(IRISDVF$Nbmutations)),
  inches = 0.08,
  col = "#2ECC40",
  frame = FALSE,
  values.rnd = 0,
  style = "e"
)

# Ajouter les étiquettes des communes selectionnées plus haut
labelLayer(Selection_Communes_RM, txt = "NOM_COM", halo = TRUE, bg = "white", r = 0.05, cex = 0.7, pos = 3, font = 3, offset = 0, overlap = TRUE)
```


---
## 3-Exploration des prix et des volumes à l'échelon des sections cadastrales


### Importer la couche des sections cadastrales (DGFip)

```{r}
Sections <- st_read(dsn = "DATA/Sectionscadastrales.shp", stringsAsFactors = FALSE)

Sections$id <- as.character(Sections$id) 
Sections <- Sections %>% mutate(codesection = id)
plot(Sections["commune"])
```

### Calculer et cartographier le prix moyen au m² par section cadastrale

```{r}
# Calcul du prix moyen au m² par section
SectionsDVF <- Sections %>% st_join(MutationsRM) %>% group_by(codesection) %>% summarise(Prixm2 = mean(prixm2))
SectionsDVF <- SectionsDVF %>% na.omit() # pour supprimer les valeurs manquantes

# Paramétrage des marges de la fenêtre pour maximiser l'emprise de la carte
par(mar = c(0, 0, 1.2, 3))

# Affichage des communes de RM et alentours en arrière-plan et centrage de la carte sur Rennes Métropole
plot(st_geometry(Communes), lwd = 0.1, col = "grey95", border = "white", bg = "#B5D0D0", xlim = c(st_bbox(IRISRM)[1], st_bbox(IRISRM)[3]), ylim = c(st_bbox(IRISRM)[2], st_bbox(IRISRM)[4])) 

# Affichage de la carte des prix au m² par section cadastrale
choroLayer(
  x = SectionsDVF,
  var = "Prixm2",
  breaks = c(1250, 1500, 2000, 2500, 3000, 3600),
  col = c("#1a9641", "#a6d96a", "#ffffbf", "#fdae61", "#d7191c"),
  add = TRUE,
  border = NA,
  legend.nodata = "Aucune mutation",
  legend.border = "white",
  legend.title.txt = "Prix moyen \nau m² (Euros)")

layoutLayer(title = "Prix moyen au m² de l'immobilier par section cadastrale dans Rennes Métropole (2014-2019)", source = "         IGN et DGFip, 2021", 
            north = TRUE, horiz = FALSE, tabtitle = TRUE,
            frame = FALSE, col = "#cdd2d4", coltitle = "#8A5543")

# Ajouter les étiquettes des communes selectionnées plus haut
labelLayer(Selection_Communes_RM, txt = "NOM_COM", halo = TRUE, bg = "white", r = 0.05, cex = 0.7, pos = 3, font = 3, offset = 0, overlap = TRUE)

```


### Calculer et cartographier le nombre de mutations par section cadastrale

```{r}
# Calcul du nombre de mutations par section
SectionsDVF <- SectionsDVF %>% mutate(Nbmutations = lengths(st_intersects(SectionsDVF, MutationsRM)))

# Paramétrage des marges de la fenêtre pour maximiser l'emprise de la carte
par(mar = c(0, 0, 1.2, 3))

# Affichage des communes de RM et alentours en arrière-plan et centrage de la carte sur Rennes Métropole
plot(st_geometry(Communes), lwd = 0.1, col = "grey95", border = "white", bg = "#B5D0D0", xlim = c(st_bbox(IRISRM)[1], st_bbox(IRISRM)[3]), ylim = c(st_bbox(IRISRM)[2], st_bbox(IRISRM)[4])) 

# Affichage des sections cadastrales
plot(st_geometry(SectionsDVF), # appel du jeu de données
     border = "grey80", # couleur de la bordure
     add = TRUE,
     lwd = 0.05) 

# Affichage des symboles proportionnels (nombre de mutations)
propSymbolsLayer(x = SectionsDVF, # appel du jeu de données
                 var = "Nbmutations", # appel de la variable à cartographier
                 col = "#2ECC40", # couleur cercles
                 border = "white",  # couleur bordure cercle
                 lwd = 0.01,
                 inches = 0.08, # Taille des cercles
                 add = TRUE,
                 legend.pos = "n",
                 ) 

layoutLayer(title = "Nombre de mutations par section cadastrale dans Rennes Métropole (2014-2019)", source = "         IGN et DGFip, 2021", north = TRUE, horiz = FALSE, tabtitle = TRUE, frame = FALSE, col = "#cdd2d4", coltitle = "#8A5543")

legendCirclesSymbols(
  pos = "bottomleft",
  title.txt = "Nombre de mutations DVF",
  title.cex = 0.8,
  cex = 1,
  border = "white",
  lwd = 0.01,
  values.cex = 0.6,
  var = c(min(IRISDVF$Nbmutations), max(IRISDVF$Nbmutations)),
  inches = 0.08,
  col = "#2ECC40",
  frame = FALSE,
  values.rnd = 0,
  style = "e"
)

# Ajouter les étiquettes des communes selectionnées plus haut
labelLayer(Selection_Communes_RM, txt = "NOM_COM", halo = TRUE, bg = "white", r = 0.05, cex = 0.7, pos = 3, font = 3, offset = 0, overlap = TRUE)
```


---
## 4-Lissage spatial (échelle métropolitaine)

### Chargement des packages nécessaires
```{r}
library(sp) # pour importer des objets ayant une composante spatiale
library(rgdal) # pour manipuler des objets ayant une composante spatiale
library(rgeos) # pour calculer des centroides
library(spdep) # pour calculer l'auto-corrélation spatiale
library(geoR) # pour calculer le semi-variogramme empirique
library(spatstat) # pour produire des surfaces lissées
library(maptools) # pour le traitement cartographique
library(raster) # pour le traitement de données matricielles
```

### Lissage spatial des prix de l'immobilier au m² calculés à partir des valeurs agrégées par IRIS

#### Calcul des distances au plus proche voisin et de l'auto-corrélation spatiale des prix au m²

```{r}
# pour convertir la couche des IRIS au format spatial object (format requis par le package geoR)
IRISDVF.sp <- as(IRISDVF, "Spatial")

# pour calculer les centroides des IRIS (indispensable pour le calcul sur les plus proches voisins)
IRISDVF.Centroids <- gCentroid(IRISDVF.sp,byid=TRUE)

# pour récupérer les données initiales des IRIS sur les centroides
IRISDVF.Centroids <- SpatialPointsDataFrame(IRISDVF.Centroids, IRISDVF.sp@data)

# Calcul sur les plus proches voisins
listPPV <- knearneigh(IRISDVF.Centroids@coords, k = 1) # pour connaître le plus proche voisin de chaque commune
PPV <- knn2nb(listPPV, row.names = IRISDVF.Centroids$code_iris) # pour convertir l'objet knn en objet nb
distPPV <- nbdists(PPV, IRISDVF.Centroids@coords) # pour connaître la distance entre plus proches voisins
print(as.data.frame(t(as.matrix(summary(unlist(distPPV))))))
hist(unlist(distPPV), breaks = 20,
     main = "Distance au plus proche voisin",
     col = "black", border = "white", xlab = "Distance", ylab = "Fréquence")

# pour convertir les IRIS en objet nb
IRIS.nb <- poly2nb(pl = IRISDVF,
                 row.names = IRISDVF$code_iris,
                 snap = 50,
                 queen = TRUE)

#calcul du test de Moran
moran.test(IRISDVF$Prixm2, listw = nb2listw(IRIS.nb))

```

#### Carte lissée des prix de l'immobilier au m² calculés à partir des valeurs agrégées par IRIS sur l'ensemble de Rennes Métropole

```{r}
# pour définir le contour de Rennes Métropole comme emprise pour le lissage (sinon le lissage est calculé sur une fenêtre rectangulaire)
EmpriseRM <- as.owin(as(IRISRM, "Spatial"))

# pour récupérer les coordonnées du centroïde des IRIS
pts <- st_coordinates(st_point_on_surface(st_geometry(IRISDVF)))

# pour créer un objet ppp (format spatstat) et y intégrer dedans l'emprise et les valeurs à lisser (prix au m²)
IRISDVF.ppp <- ppp(pts[,1], pts[,2], window = EmpriseRM, marks = IRISDVF$Prixm2)

# pour calculer la surface lissée (rayon lissage : 1 km et resolution spatiale de l'image : 1 ha)
cartelissee <- Smooth(IRISDVF.ppp, sigma = 1000, weights = IRISDVF.ppp$marks, eps=100)

# Conversion de la surface lissée au format raster
cartelissee.raster <- raster(cartelissee, crs = st_crs(IRISDVF)[[2]])

# Paramétrage des marges de la fenêtre pour maximiser l'emprise de la carte
par(mar = c(0, 0, 1.2, 2))

# pour afficher la surface lissée et définir l'habillage
# Calcul des seuils
bks <- unique(getBreaks(values(cartelissee.raster), method = "q6"))

# Création d'une palette de couleurs (double gradation harmonique)
cols <- c("#2a9c4e", "#77c35c", "#c4e687", "#ffffc0", "#fec981", "#dc292c")

# Affichage des communes de RM et alentours en arrière-plan et centrage de la carte sur Rennes Métropole
plot(st_geometry(Communes), lwd = 0.1, col = "grey95", border = "white", bg = "#B5D0D0", xlim = c(st_bbox(IRISRM)[1], st_bbox(IRISRM)[3]), ylim = c(st_bbox(IRISRM)[2], st_bbox(IRISRM)[4]))

# Affichage de la carte lissée et du contour des IRIS
plot(cartelissee.raster, breaks = bks, col=cols, add = T, legend=F)
plot(IRISDVF$geometry, border = "grey60", lwd = 0.05, lty=3, add = T)

legendChoro(
  pos = "bottomleft",
  title.txt = "Prix moyen \nau m² (Euros)",
  breaks = bks, 
  nodata = FALSE,
  values.rnd = -1,
  border = "white",
  col = cols
)

layoutLayer(title = "Prix moyen au m² de l'immobilier dans Rennes Métropole (2014-2019)", 
            author = "          Sources : IGN et DGFip - Rayon : 1000 m, résolution : 1 ha - A partir des IRIS", 
            scale = 5, frame = TRUE, col = "#cdd2d4", coltitle = "#8A5543", 
            north(pos = "topleft"), tabtitle=TRUE, horiz = FALSE)

# Ajouter les étiquettes des communes sélectionnées plus haut
labelLayer(Selection_Communes_RM, txt = "NOM_COM", halo = TRUE, bg = "white", r = 0.05, cex = 0.7, pos = 3, font = 3, offset = 0)
```

### Lissage spatial du prix de l'immobilier au m² calculés à partir des valeurs agrégées par section cadastrale

#### Calcul des distances au plus proche voisin et de l'auto-corrélation spatiale des prix au m²

```{r}
# pour convertir la couche des sections au format spatial object (format requis par le package geoR)
SectionsDVF.sp <- as(SectionsDVF, "Spatial")

# pour calculer les centroides des sections (indispensable pour le calcul sur les plus proches voisins)
SectionsDVF.Centroids <- gCentroid(SectionsDVF.sp,byid=TRUE)

# pour récupérer les données initiales des sections sur les centroides
SectionsDVF.Centroids <- SpatialPointsDataFrame(SectionsDVF.Centroids, SectionsDVF.sp@data)

# Calcul sur les plus proches voisins
listPPV <- knearneigh(SectionsDVF.Centroids@coords, k = 1) # pour connaître le plus proche voisin de chaque commune
PPV <- knn2nb(listPPV, row.names = SectionsDVF.Centroids$codesection) # pour convertir l'objet knn en objet nb
distPPV <- nbdists(PPV, SectionsDVF.Centroids@coords) # pour connaître la distance entre plus proches voisins
print(as.data.frame(t(as.matrix(summary(unlist(distPPV))))))
hist(unlist(distPPV), breaks = 20,
     main = "Distance au plus proche voisin",
     col = "black", border = "white", xlab = "Distance", ylab = "Fréquence")

# pour convertir les sections en objet nb
Section.nb <- poly2nb(pl = SectionsDVF,
                 row.names = SectionsDVF$codesection,
                 snap = 50,
                 queen = TRUE)

#calcul du test de Moran
moran.test(SectionsDVF$Prixm2, listw = nb2listw(Section.nb))

```

#### Carte lissée des prix de l'immobilier au m² calculés à partir des valeurs agrégées par section cadastrale sur l'ensemble de Rennes Métropole

```{r}
# pour définir le contour de Rennes Métropole comme emprise pour le lissage (sinon le lissage est calculé sur une fenêtre rectangulaire)
EmpriseRM <- as.owin(as(IRISRM, "Spatial"))

# pour récupérer les coordonnées du centroïde des sections
pts <- st_coordinates(st_point_on_surface(st_geometry(SectionsDVF)))

# pour créer un objet ppp (format spatstat) et y intégrer dedans l'emprise et les valeurs à lisser (prix au m²)
Sections.ppp <- ppp(pts[,1], pts[,2], window = EmpriseRM, marks = SectionsDVF$Prixm2)

# pour calculer la surface lissée (rayon lissage : 1 km et resolution spatiale de l'image : 1 ha)
cartelissee <- Smooth(Sections.ppp, sigma = 1000, weights = Sections.ppp$marks, eps=100)

# Conversion de la surface lissée au format raster
cartelissee.raster <- raster(cartelissee, crs = st_crs(SectionsDVF)[[2]])

# Paramétrage des marges de la fenêtre pour maximiser l'emprise de la carte
par(mar = c(0, 0, 1.2, 2))

# pour afficher la surface lissée et définir l'habillage
# Calcul des seuils
bks <- unique(getBreaks(values(cartelissee.raster), method = "q6"))

# Création d'une palette de couleurs (double gradation harmonique)
cols <- c("#2a9c4e", "#77c35c", "#c4e687", "#ffffc0", "#fec981", "#dc292c")

# Affichage des communes de RM et alentours en arrière-plan et centrage de la carte sur Rennes Métropole
plot(st_geometry(Communes), lwd = 0.1, col = "grey95", border = "white", bg = "#B5D0D0", xlim = c(st_bbox(SectionsDVF)[1], st_bbox(SectionsDVF)[3]), ylim = c(st_bbox(SectionsDVF)[2], st_bbox(SectionsDVF)[4]))

# Affichage de la carte lissée et du contour des sections
plot(cartelissee.raster, breaks = bks, col=cols, add = T, legend=F)
plot(SectionsDVF$geometry, border = "grey60", lwd = 0.05, lty=3, add = T)

legendChoro(
  pos = "bottomleft",
  title.txt = "Prix moyen \nau m² (Euros)",
  breaks = bks, 
  nodata = FALSE,
  values.rnd = -1,
  border = "white",
  col = cols
)

layoutLayer(title = "Prix moyen au m² de l'immobilier dans Rennes Métropole (2014-2019)", 
            author = "      Sources : IGN, DGFip - Rayon : 1000 m, résolution : 1 ha - A partir des sections", 
            scale = 5, frame = TRUE, col = "#cdd2d4", coltitle = "#8A5543", 
            north(pos = "topleft"), tabtitle=TRUE, horiz = FALSE)

# Ajouter les étiquettes des communes sélectionnées plus haut
labelLayer(Selection_Communes_RM, txt = "NOM_COM", halo = TRUE, bg = "white", r = 0.05, cex = 0.7, pos = 3, font = 3, offset = 0)
```

### Lissage spatial des prix de l'immobilier au m² calculés directement à partir des mutations immobilières sur l'ensemble de Rennes Métropole

#### Calcul de la distance au plus proche voisin et de l'auto-corrélation spatiale des prix au m²

```{r}
# pour convertir les mutations en objet spatial (format requis par le package geoR)
MutationsRM.sp <- as(MutationsRM, "Spatial")

# Calcul sur les plus proches voisins
listPPV <- knearneigh(MutationsRM.sp@coords, k = 1) # pour connaître le plus proche voisin de chaque mutation
PPV <- knn2nb(listPPV, row.names = MutationsRM.sp$id) # pour convertir l'objet knn en objet nb
distPPV <- nbdists(PPV, MutationsRM.sp@coords) # pour connaître la distance entre plus proches voisins
print(as.data.frame(t(as.matrix(summary(unlist(distPPV))))))
hist(unlist(distPPV), nclass = 200,
     main = "Distance au plus proche voisin",
     col = "black", border = "white", xlab = "Distance", ylab = "Fréquence", xlim = c(0,150))

#calcul du test de Moran
moran.test(MutationsRM.sp$prixm2, listw = nb2listw(PPV))
```


#### Carte lissée des prix de l'immobilier au m² calculés directement à partir des mutations immobilières sur l'ensemble de Rennes Métropole

```{r}
# Si nécessaire,  convertir les mutations en objet spatial (format requis par le package geoR)
MutationsBZHsp <- as(MutationsBZH, "Spatial")

# pour définir comme emprise pour le lissage, l'étendue de Rennes métropole avec une zone tampon de 0,5 km autour afin d'englober les mutations limitrophes et réduire l'effet de bord
EmpriseRM.ZT <- as.owin(as(st_union(st_buffer(IRISRM, 500)), "Spatial"))

# pour créer un objet ppp (format spatstat) et y intégrer dedans l'emprise et les valeurs à lisser (prix moyen au m²)
# NB : on considère dans un premier temps les mutations sur l'ensemble de la Bretagne, et on ne retient que celles qui sont incluses dans l'emprise créée à l'étape précédente
MutationsBZHsp.ppp <- ppp(MutationsBZHsp@coords[,1], MutationsBZHsp@coords[,2], window = EmpriseRM.ZT, marks = MutationsBZHsp$prixm2)

# pour calculer la surface lissée (rayon lissage : 0,5 km et résolution spatiale de l'image : 1 ha) --> calcul long
cartelissee <- Smooth(MutationsBZHsp.ppp, sigma = 500, weights = MutationsBZHsp.ppp$marks, eps=100)

# Conversion de la surface lissée au format raster
cartelissee.raster <- raster(cartelissee, crs = st_crs(MutationsRM)[[2]])

# découpage de la surface lissée sur l'emprise Rennes Métropole
cartelissee.raster <- mask(cartelissee.raster, IRISRM) 

# Paramétrage des marges de la fenêtre pour maximiser l'emprise de la carte
par(mar = c(0, 0, 1.2, 2))

# pour afficher la surface lissée et définir l'habillage de la carte
# Calcul des seuils
bks <- unique(getBreaks(values(cartelissee.raster), method = "q6"))

# Création d'une palette de couleurs (double gradation harmonique)
cols <- c("#2a9c4e", "#77c35c", "#c4e687", "#ffffc0", "#fec981", "#dc292c")

# Affichage des communes de RM et alentours en arrière-plan et centrage de la carte sur Rennes Métropole
plot(st_geometry(Communes), lwd = 0.1, col = "grey95", border = "white", bg = "#B5D0D0", xlim = c(st_bbox(IRISRM)[1], st_bbox(IRISRM)[3]), ylim = c(st_bbox(IRISRM)[2], st_bbox(IRISRM)[4]))

# Affichage de la carte lissée et du contour des IRIS
plot(cartelissee.raster, breaks = bks, col=cols, add = T, legend=F)
plot(IRISRM$geometry, border = "grey60", lwd = 0.05, lty=3, add = T)

legendChoro(
  pos = "bottomleft",
  title.txt = "Prix moyen \nau m² (Euros)",
  breaks = bks, 
  nodata = FALSE,
  values.rnd = -1,
  border = "white",
  col = cols
)

layoutLayer(title = "Prix moyen au m² de l'immobilier dans Rennes Métropole (2014-2019)", 
            author = "      Sources : IGN, DGFip - Rayon : 500 m, résolution : 1 ha - A partir des mutations", 
            scale = 5, frame = TRUE, col = "#cdd2d4", coltitle = "#8A5543", 
            north(pos = "topleft"), tabtitle=TRUE, horiz = FALSE)

# Ajouter les étiquettes des communes sélectionnées plus haut
labelLayer(Selection_Communes_RM, txt = "NOM_COM", halo = TRUE, bg = "white", r = 0.05, cex = 0.7, pos = 3, font = 3, offset = 0)
```


---

## 5-Elaboration d'une typologie à partir d'indicateurs immobiliers et cartographie des sous-marchés associés (échelle métropolitaine)

### Chargement du package nécessaire

```{r}
library(cluster)
```

### Transfert d'attributs des IRIS vers les mutations (jointure spatiale)  
```{r}
MutationsIRIS <- MutationsRM %>% st_join(IRISRM)
MutationsIRIS <- as.data.frame(MutationsIRIS)

```

### Tableau récapitulatif des variables par IRIS à soumettre à la CAH
```{r}
IRISDVFClassif1 <- MutationsIRIS %>% group_by(code_iris) %>%
  summarise(Nbtransactions = n(),
            Prixmoyen = mean(prix),
            Prixm2moyen = mean(prixm2),
            Surfacemoyenne = mean(surface),
            PropMaison = length(type[type=="Maison"])/Nbtransactions*100,
            PropAppart = length(type[type=="Appartement"])/Nbtransactions*100)

IRISDVFClassif <- data.frame(IRISDVFClassif1[, c("Nbtransactions", "Prixmoyen", "Prixm2moyen", "Surfacemoyenne", "PropMaison", "PropAppart")])
```

### Centrage et réduction des variables
```{r}
IRISDVFClassifscale <- scale(IRISDVFClassif)
```

### Mise en oeuvre de la CAH
```{r}
CAHIRIS <- agnes(IRISDVFClassifscale,
                     metric = "euclidean",
                     method = "ward")
```

### Graphiques des gains d'inertie inter-classe
```{r}
sortedHeight<- sort(CAHIRIS$height,decreasing= TRUE)
relHeight<-sortedHeight/ sum(sortedHeight)*100
cumHeight<- cumsum(relHeight)

barplot(relHeight[1:30],names.arg=seq(1, 30, 1),col= "black",border= "white",xlab= "Noeuds",ylab= "Part de l'inertie totale (%)")
barplot(cumHeight[1:30],names.arg=seq(1, 30, 1),col= "black",border= "white",xlab= "Nombre de classes",ylab= "Part de l'inertie totale (%)")
```

### Arbre de classification hiérarchique (dendrogramme)
```{r}
dendroCSP <- as.dendrogram(CAHIRIS)
plot(dendroCSP, leaflab = "none")
```

### Partition (en n classes)
```{r}
clusIRIS <- cutree(CAHIRIS, k = 5)
IRISCluster <- as.data.frame(IRISDVFClassif1)
IRISCluster$CLUSIMMO <- factor(clusIRIS,
                                   levels = 1:5,
                                   labels = paste("Classe", 1:5))
```

### Tableau récapitulatif des groupes
```{r}
RecapCAHIRIS <- IRISCluster %>% group_by(CLUSIMMO) %>% summarise(NB= n(), NbTransac = mean(Nbtransactions), Prixmoyen = mean(Prixmoyen), Prixm2 = mean(Prixm2moyen), Surface=mean(Surfacemoyenne), PropMaison = mean(PropMaison), PropAppart= mean(PropAppart))

print(RecapCAHIRIS)
```

### Graphique des écarts à la moyenne
```{r}
SyntheseCAHIRIS <- RecapCAHIRIS %>% mutate(
  nbtransacmoy = mean(IRISDVFClassif$Nbtransactions),
  surfacemoy = mean(IRISDVFClassif$Surfacemoyenne),
  prixmoy = mean(IRISDVFClassif$Prixmoyen),
  prixm2moyen = mean(IRISDVFClassif$Prixm2moyen),
  propmaisonmoyen = mean(IRISDVFClassif$PropMaison),
  propappartmoyen = mean(IRISDVFClassif$PropAppart),
  NbMutations=(NbTransac- nbtransacmoy)/nbtransacmoy*100,
  Prix=(Prixmoyen- prixmoy)/prixmoy*100,
  Prixm2=(Prixm2- prixm2moyen)/prixm2moyen*100,
  Surface=(Surface- surfacemoy)/surfacemoy*100,
  PropMaison=(PropMaison- propmaisonmoyen)/propmaisonmoyen*100,
  PropAppart=(PropAppart- propappartmoyen)/propappartmoyen*100
)

# pour créer un nouveau tableau avec une sélection de colonnes à faire apparaître sur le graphique
SyntheseCAHIRIS <- data.frame(SyntheseCAHIRIS[, c("CLUSIMMO", "NbMutations", "Surface", "Prix", "Prixm2", "PropMaison", "PropAppart")])

gather <- SyntheseCAHIRIS %>% gather(key=variable, value= "value", NbMutations:PropAppart)

ggplot(gather, aes(x=variable, y=value, fill=CLUSIMMO)) +
  geom_bar(stat = "identity") +
  coord_flip() +
  scale_fill_manual(values=c("#2A9D8F","#264653","#fcbf49","#e45c3a","#0074D9")) +
  ylab("Variation par rapport à la moyenne métropolitaine (%)") +
  theme_bw() +
  theme(legend.position = "none") +
  facet_wrap(~CLUSIMMO, ncol = 1)
```

### Carte de la typologie montrant les sous-marchés immobiliers métropolitains
```{r}
# pour joindre le résultat de la typologie dans la couche des IRIS
IRISDVFCAH <- left_join(IRISRM, IRISCluster, by= "code_iris")

# Paramétrage des marges de la fenêtre pour maximiser l'emprise de la carte
par(mar = c(0, 0, 1.2, 2))

# pour afficher la surface lissée et définir l'habillage de la carte
# Calcul des seuils
bks <- unique(getBreaks(values(cartelissee.raster), method = "q6"))

# Création d'une palette de couleurs (double gradation harmonique)
cols <- c("#2a9c4e", "#77c35c", "#c4e687", "#ffffc0", "#fec981", "#dc292c")

# Affichage des communes de RM et alentours en arrière-plan et centrage de la carte sur Rennes Métropole
plot(st_geometry(Communes), lwd = 0.1, col = "grey95", border = "white", bg = "#B5D0D0", xlim = c(st_bbox(IRISRM)[1], st_bbox(IRISRM)[3]), ylim = c(st_bbox(IRISRM)[2], st_bbox(IRISRM)[4]))

# Affichage de la typologie et du contour des IRIS
typoLayer(
  x = IRISDVFCAH,
  var="CLUSIMMO",
  col = c("#2A9D8F","#264653","#fcbf49","#e45c3a","#0074D9"),
  lwd = 0.05,
  border = "grey70",
  legend.values.order = c("Classe 1",
                          "Classe 2",
                          "Classe 3",
                          "Classe 4",
                          "Classe 5"),
  legend.pos = "bottomleft",
  legend.title.txt = "Sous-marchés \nimmobiliers",
  legend.nodata = "Aucune mutation",
  add = TRUE)

layoutLayer(title = "Sous-marchés immobiliers dans Rennes Métropole à l'échelon des IRIS (2014-2019)", 
            author = "          Sources : IGN et DGFip - Typologie obtenue par CAH", 
            scale = 5, frame = TRUE, col = "#cdd2d4", coltitle = "#8A5543", 
            north(pos = "topleft"), tabtitle=TRUE, horiz = FALSE)

# Ajouter les étiquettes des communes sélectionnées plus haut
labelLayer(Selection_Communes_RM, txt = "NOM_COM", halo = TRUE, bg = "white", r = 0.05, cex = 0.7, pos = 3, font = 3, offset = 0)

```

### Export des IRIS contenant la typologie issue de la CAH au format geopackage (pour une utilisation dans un SIG par exemple)

```{r, include=TRUE}
st_write(IRISDVFCAH, "Exports/IRISDVFCAH.gpkg", append = FALSE)
```


---
## 6-Lissage spatial des prix au m² des appartements entre 2014 et 2019 : zoom sur la commune de Rennes

### Préparation du jeu de données

```{r}
# pour ne garder que la commmune de Rennes
CommuneRennes <- Communes %>% filter(NOM_COM == "Rennes")

# pour ne garder que les mutations correspondant à des appartements dans Rennes Métropole (dans un premier temps)
MutationsApptRM <- MutationsRM %>% filter(type == "Appartement")

# pour créer une liste d'années, dans l'ordre
MutationsApptRM<-MutationsApptRM[order(MutationsApptRM$annee, decreasing = FALSE), ]
ListAnnees <- unique(MutationsApptRM$annee)
```

### Série de cartes lissées montrant l'évolution des prix au m² des appartements calculés directement à partir des mutations, à Rennes entre 2014 et 2019
```{r eval=FALSE, message=FALSE, warning=FALSE, include=FALSE}
# pour définir comme emprise pour le lissage, l'étendue de Rennes avec une zone tampon de 0,5 km autour afin d'englober les mutations limitrophes et réduire l'effet de bord
EmpriseRennes <- as.owin(as(st_union(st_buffer(CommuneRennes, 500)), "Spatial"))

# Paramétrage des marges pour insérer le titre général et les titres de chaque carte
par(oma=c(3.5,0,3,0)+0.1, mar = c(0, 0.5, 1.2, 0.5))

plot.new()

# pour découper la fenêtre en 2 lignes et 3 colonnes (6 années)
par(mfrow=c(2,3))

# boucle pour produire et cartographier une surface lissée par année
for (i in ListAnnees){
  
  # Nommage des dataframes
  name<- paste("PrixM2_Appt_Rennes",i, sep="-")
  
  # Recuperation des jeux de données par année
  b<-assign(name, MutationsApptRM[which(MutationsApptRM$annee == i),]) # La fonction assign permet d'assigner un nom à une valeur/un element
  fichier<-get(name)
  
  # pour récupérer les coordonnées des mutations
  pts <- st_coordinates(st_geometry(fichier))
  
  # pour creer un objet ppp (format spatstat) et y integrer dedans l'emprise et les valeurs a lisser (prix au m²)
  fichier.ppp <- ppp(pts[,1], pts[,2], window = EmpriseRennes, marks = fichier$prixm2)
  
  # pour calculer la surface lissee (rayon lissage : 500 m et resolution spatiale de l'image : 1000m²)
  cartelissee <- Smooth(fichier.ppp, sigma = 500, weights = fichier.ppp$marks, eps=sqrt(1000))
  
  # Conversion de la surface lissée au format raster
  cartelissee.raster <- raster(cartelissee, crs = st_crs(CommuneRennes)[[2]])
  cartelissee.raster <- mask(cartelissee.raster, CommuneRennes) # découpage sur emprise Rennes
  
  # Définition manuelle des seuils
  bks <- c(cellStats(cartelissee.raster, stat='min'), 1500, 2000, 2200, 2400, 2800, 3000, 3200, 3500, cellStats(cartelissee.raster, stat='max'))
  
  # Création d'une palette de couleurs (double gradation harmonique)
  cols <- c("#fff7f3", "#fee1d3", "#fcbea5", "#fc9777", "#fb7050", "#f14432", "#d42020", "#ad1016", "#67000d", "#480000")
  
  # reclassification de la surface lissée
  cartelissee.reclass <- cut(cartelissee.raster, breaks = bks)
  
  # vectorisation de la surface reclassée (calcul un peu long)
  cartelissee.vecteur <- as(rasterToPolygons(cartelissee.reclass, n=4, na.rm=TRUE, digits=12, dissolve=TRUE), "sf")
  
  # Tracer la carte
  plot(st_geometry(CommuneRennes), border = "white", bg= "grey90")
  
  typoLayer(
    x = cartelissee.vecteur,
    var="layer",
    col = cols,
    lwd = 0.1,
    border = cols,
    legend.pos = "n",
    add = TRUE)
  
  plot(IRISDVF$geometry, border = "white", lty = 3, add=TRUE)
  
  title(main =paste("",i, sep=""))
  
}

barscale(
  lwd = 1.5,
  cex = 0.6,
  pos = "bottomleft",
  style = "pretty",
  unit = "m"
)

north(pos = "topleft")

# Pour afficher le titre principal et la source
mtext("Prix moyen en Euros au m² des appartements à Rennes de 2014 à 2019", cex = 1.3, side=3,line=1,adj=0.5,outer=TRUE)
mtext("   Sources : IGN et DGFip - Rayon de lissage : 500 m et résolution : 1000 m²", side=1, line=1, adj=0, cex=0.6, outer=TRUE)

# Overlay the entire figure region with a new, single plot. 
par(fig = c(0, 1, 0, 1), oma = c(0, 0, 0, 0), mar = c(0, 0, 0, 0), new = TRUE)
plot(0, 0, type = "n", bty = "n", xaxt = "n", yaxt = "n")

# Then call legend with the location ("bottom", "right", "bottomright", etc.)
legend("bottom", 
       (legendChoro(
         pos = "bottomright",
         title.txt = "",
         breaks = bks, 
         nodata = FALSE,
         values.rnd = -1,
         col = cols,
         border = "white",
         horiz = TRUE
       )),
       xpd = TRUE
)

```
