# Example of a full species distribution modelling workflow

## Environmental data

```@example test
#| echo: false
if !haskey(ENV, "RASTERDATASOURCES_PATH")
    ENV["RASTERDATASOURCES_PATH"] = ".";
end
using CairoMakie
CairoMakie.activate!(type = "png")
```

```@example test
using Rasters, RasterDataSources, ArchGDAL, NaturalEarth, DataFrames
bio = RasterStack(WorldClim{BioClim}, (1,12))
countries = naturalearth("ne_10m_admin_0_countries") |> DataFrame
australia = subset(countries, :NAME => ByRow(==("Australia"))).geometry
bio_aus = Rasters.trim(mask(bio; with = australia)[X = 110 .. 156, Y = -45 .. -10])
```

## Environmental data
```@example test
using CairoMakie
Rasters.rplot(bio_aus)
```

## Occurrence data
```@example test
using GBIF2, SpeciesDistributionModels
sp = species_match("Eucalyptus regnans")
occurrences_raw = occurrence_search(sp; year = (1970,2000), country = "AU", hasCoordinate = true, limit = 2000)
occurrences = thin(occurrences_raw.geometry, 5000)

```
## Background points
```@example test
using StatsBase
bg_indices = sample(findall(boolmask(bio_aus)), 500)
bg_points = DimPoints(bio_aus)[bg_indices]
fig, ax, pl = plot(bio_aus.bio1)
scatter!(ax, occurrences; color = :red)
scatter!(ax, bg_points; color = :grey)
fig
```

## Handling data
```@example test
using SpeciesDistributionModels
p_data = extract(bio_aus, occurrences; skipmissing = true)
bg_data = bio_aus[bg_indices]
data = sdmdata(p_data, bg_data; resampler = CV(nfolds = 3))
```

## Fitting an ensemble
```@example test
using Maxnet: MaxnetBinaryClassifier
using EvoTrees: EvoTreeClassifier
using MLJGLMInterface: LinearBinaryClassifier
models = (
  maxnet = MaxnetBinaryClassifier(),
  brt = EvoTreeClassifier(),
  glm = LinearBinaryClassifier()
)

ensemble = sdm(data, models)
```

## Evaluating an ensemble
```@example test
import SpeciesDistributionModels as SDM
ev = SDM.evaluate(ensemble; measures = (; auc, accuracy))
```

## Predicting
```@example test
pred = SDM.predict(ensemble, bio_aus; reducer = mean)
plot(pred; colorrange = (0,1))
```

## Understanding the model
```@example test
expl = SDM.explain(ensemble; method = ShapleyValues(8))
interactive_response_curves(expl)
```