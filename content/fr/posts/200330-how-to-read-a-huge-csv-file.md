---
title: "[.NET] Comment lire un très gros fichier csv"
slug: "dotnet comment lire un tres gros fichier csv"
date: 2020-03-30
description: ""
draft: false
tags:
  - dotnet
  - csharp
  - algorithm
  - performance
  - benchmark
series:
  - benchmark
categories:
  - dotnet
libraries:
  - chart
image: images/logo/dotnetcore.png
---

Le traitement d'un fichier csv de plusieurs Go peut vite être coûteux en terme de performance.
Pour rappel un fichier csv n'est pas seulement un format qui sépare ses colonnes par un caractère, mais c'est aussi:

- des entêtes présentes ou non,
- des colonnes parfois inexistantes,
- des lignes vides,
- des guillemets pour représenter une colonne,
- des guillemets dans des guillemets pour représenter des guillemets dans une colonne…
- …

Bref, la liste est encore longue et ça peut vite devenir un vrai casse tête de gérer tous les cas.
A cela s'ajoute qu'il faut la plupart du temps lier les colonnes à des objets de notre code, que dans le cas d'un très gros fichier il n'est pas envisageable de charger le fichier en mémoire et qu'il faudra lire à même le stream ce qui peut apporter d'autres problématiques _(heureusement en .NET cette dernière problématique reste extrêmement simple à solutionner)_ cela peut devenir relativement compliqué.

Bien entendu d'autres se sont pris la tête là-dessus, c'est pour cette raison qu'il existe un grand nombre de framework de lecture/écriture de fichier csv qui gère la totalité des problématiques liées à ce format.

## TextFieldParser

Le framework .NET propose une solution de base avec **TextFieldParser**.

{{< codes SampleCsvParser Foo >}}
{{< code >}}

```csharp
[Benchmark]
public void TextFieldParser()
{
    using var streamReader = new StreamReader(_csvFilePath);
    TextFieldParser parser = new TextFieldParser(_csvFilePath)
    {
        HasFieldsEnclosedInQuotes = true,
        Delimiters = new[] { "," }
    };
    string[] fields;
    while ((fields = parser.ReadFields()) != null)
    {
        Foo.CreateFromFields(fields);
        // ...
    }
}
```

{{< /code >}}

{{< code >}}

```csharp
public class Foo
{
    public string Prop0 { get; set; }
    public string Prop1 { get; set; }
    public string Prop2 { get; set; }

    public static Foo CreateFromFields(string[] fields)
    {
        return new Foo
        {
            Prop0 = fields[0],
            Prop1 = fields[1],
            Prop2 = fields[2],
        };
    }
}
```

{{< /code >}}
{{< /codes >}}

## CsvHelper

sources: [https://github.com/JoshClose/CsvHelper](https://github.com/JoshClose/CsvHelper)

{{< codes SampleCsvParser FooMapping >}}
{{< code >}}

```csharp
[Benchmark]
public void CsvHelper()
{
    using var streamReader = new StreamReader(_csvFilePath);
    var csvconfig = new CsvConfiguration(CultureInfo.CurrentCulture) { Delimiter = ",", HasHeaderRecord = false };
    csvconfig.RegisterClassMap<CsvHelperFooMapping>();
    var csv = new CsvReader(streamReader, csvconfig);
    using var records = csv.GetRecords<Foo>().GetEnumerator();
    while (records.MoveNext())
    {
        // ...
    }
}
```

{{< /code >}}

{{< code >}}

```csharp
public sealed class CsvHelperFooMapping: ClassMap<Foo>
{
    public CsvHelperFooMapping()
    {
        Map(x => x.Prop0).Index(0);
        Map(x => x.Prop1).Index(1);
        Map(x => x.Prop2).Index(2);
    }
}
```

{{< /code >}}
{{< /codes >}}

## TinyCsvParser

sources: [https://github.com/bytefish/TinyCsvParser](https://github.com/bytefish/TinyCsvParser)

{{< codes SampleCsvParser FooMapping >}}
{{< code >}}

```csharp
public void TinyCsvParser()
{
    var csvParserOptions = new CsvParserOptions(false, ',', Environment.ProcessorCount, false);
    var csvMapper = new TinyFooMapping();
    var csvParser = new CsvParser<Foo>(csvParserOptions, csvMapper);
    using var records = csvParser.ReadFromFile(_csvFilePath, Encoding.UTF8).GetEnumerator();
    while (records.MoveNext())
    {
        // ...
    }
}
```

{{< /code >}}

{{< code >}}

```csharp
public sealed class TinyFooMapping : CsvMapping<Foo>
{
    public TinyFooMapping()
    {
        MapProperty(0, x => x.Prop0);
        MapProperty(1, x => x.Prop1);
        MapProperty(2, x => x.Prop2);
    }
}
```

{{< /code >}}
{{< /codes >}}

## Résultat (100 000 lignes + ssd)

| Method            |            Mean |                                           % |
| ----------------- | --------------: | ------------------------------------------: |
| TextFieldParser   |     6,617.43 ms |                                          0% |
| CsvHelper         |     4,018.37 ms |     <span style="color: green">-39 %</span> |
| **TinyCsvParser** | **1,062.29 ms** | **<span style="color: green">-84 %</span>** |

Doit-on d'office exclure **CsvHelper**?
Non, comme toujours en développement rien n'est blanc ou noir.
Dans ce cas spécifique _(un csv propre, une configuration de base et une lecture d'un gros fichier)_ **TinyCsvParser** est préférable en revanche dans d'autres scénarii **CsvHelper** apporte des fonctionnalités très intéressantes _(comme l'auto-mapping)_ qui peuvent faire gagner un précieux temps de développement et de maintenance.
Bref cela dépend de la situation comme toujours.

{{< boxmd >}}
**Mais comment expliquer de tels décalages ?**
{{< /boxmd >}}

Principalement cela est dû au traitement des lignes du CSV pour gérer tous les cas et la création des entités.
Nous avons tendance à se ruer sur des framework pour des choses simples, ces framework sont là pour gérer tous les cas ce qui apporte un confort de développement, de la fiabilité mais malheureusement aussi une lourdeur dans les traitements.
J'ai récemment dû parser un fichier de 30Go "propre" _(virgule + parfois des guillemets)_ dont je connais parfaitement le format, et j'ai voulu voir combien coûte "la gestion de tous les cas".

## Solution personnalisée

{{< codes SampleCsvParser ExtractFields >}}
{{< code >}}

```csharp
[Benchmark]
public void Custom()
{
    using var streamReader = new StreamReader(_csvFilePath);
    string line;
    while ((line = streamReader.ReadLine()) != null)
    {
        Foo.CreateFromCsvLine(line);
        // ...
    }
}
```

{{< /code >}}

{{< code >}}

```csharp
public class Foo
{
  ...

  public static Foo CreateFromCsvLine(string line)
  {
      return CreateFromFields(ExtractFields(line, 3));
  }

  private static string[] ExtractFields(string line, int propertyCount)
  {
      var result = new string[propertyCount];
      var index = 0;
      bool isInQuotes = false;
      var chars = line.ToCharArray();
      StringBuilder str = new StringBuilder(string.Empty);
      foreach (var t in chars)
      {
          if (t == '"')
          {
              isInQuotes = !isInQuotes;
          }
          else if (t == ',' && !isInQuotes)
          {
              result[index++] = str.ToString();
              str.Clear();
          }
          else
          {
              str.Append(t);
          }
      }
      result[index] = str.ToString();

      return result;
  }
}
```

{{< /code >}}
{{< /codes >}}

L'algorithme d'analyse des lignes csv est simple et limitée mais correspond à mon besoin.
Et voici le résultat :

| Method            |            Mean |                                           % |
| ----------------- | --------------: | ------------------------------------------: |
| TextFieldParser   |     6,617.43 ms |                                          0% |
| CsvHelper         |     4,018.37 ms |     <span style="color: green">-39 %</span> |
| **TinyCsvParser** | **1,062.29 ms** | **<span style="color: green">-84 %</span>** |
| Custom            |     1,083.97 ms |     <span style="color: green">-83 %</span> |

Comment **TinyCsvParser** peut-il encore être plus performant ? Simplement car ce framework utilise de base le parallélisme. Nous allons en faire de même pour la science.

## Solution personnalisée avec parallélisme

{{< codes SampleCsvParser StreamReaderEnumerator >}}
{{< code >}}

```csharp
[Benchmark]
public void CustomParallel()
{
    using var streamReader = new StreamReader(_csvFilePath);
    var enumerator = new StreamReaderEnumerable(streamReader);
    var po = new ParallelOptions
    {
        // just for demo
        MaxDegreeOfParallelism = Environment.ProcessorCount
    };

    var action = new Action<string>(line =>
    {
        Foo.CreateFromCsvLine(line);
        // ...
    });

    Parallel.ForEach(enumerator, po, action);
}
```

{{< /code >}}

{{< code >}}

```csharp
/// <summary>
/// This method comes from <see href="!:https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.ienumerable-1.etenumerator">msdn</see>
/// </summary>
public class StreamReaderEnumerable : IEnumerable<string>
{
    private readonly StreamReader _sr;
    public StreamReaderEnumerable(StreamReader streamReader)
    {
        _sr = streamReader;
    }

    // Must implement GetEnumerator, which returns a new StreamReaderEnumerator.
    public IEnumerator<string> GetEnumerator()
    {
        return new StreamReaderEnumerator(_sr);
    }

    // Must also implement IEnumerable.GetEnumerator, but implement as a private method.
    private IEnumerator GetEnumerator1()
    {
        return GetEnumerator();
    }

    IEnumerator IEnumerable.GetEnumerator()
    {
        return GetEnumerator1();
    }
}

/// <summary>
/// This method comes from <see href="!:https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.ienumerable-1.etenumerator">msdn</see>
/// </summary>
public class StreamReaderEnumerator : IEnumerator<string>
{
    private readonly StreamReader _sr;

    public StreamReaderEnumerator(StreamReader streamReader)
    {
        _sr = streamReader;
    }

    private string _current;
    // Implement the IEnumerator(T).Current publicly, but implement
    // IEnumerator.Current, which is also required, privately.
    public string Current
    {

        get
        {
            if (_sr == null || _current == null)
            {
                throw new InvalidOperationException();
            }

            return _current;
        }
    }

    private object Current1 => this.Current;

    object IEnumerator.Current => Current1;

    // Implement MoveNext and Reset, which are required by IEnumerator.
    public bool MoveNext()
    {
        _current = _sr.ReadLine();

        return _current != null;
    }

    public void Reset()
    {
        _sr.DiscardBufferedData();
        _sr.BaseStream.Seek(0, SeekOrigin.Begin);
        _current = null;
    }

    // Implement IDisposable, which is also implemented by IEnumerator(T).
    private bool _disposedValue;
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    protected virtual void Dispose(bool disposing)
    {
        if (!_disposedValue)
        {
            if (disposing)
            {
                // Dispose of managed resources.
            }
            _current = null;
            if (_sr != null)
            {
                _sr.Close();
                _sr.Dispose();
            }
        }

        _disposedValue = true;
    }

    ~StreamReaderEnumerator()
    {
        Dispose(false);
    }
}
```

{{< /code >}}
{{< /codes >}}

| Method             |          Mean |                                           % |
| ------------------ | ------------: | ------------------------------------------: |
| TextFieldParser    |   6,617.43 ms |                                          0% |
| CsvHelper          |   4,018.37 ms |     <span style="color: green">-39 %</span> |
| TinyCsvParser      |   1,062.29 ms |     <span style="color: green">-84 %</span> |
| Custom             |   1,083.97 ms |     <span style="color: green">-83 %</span> |
| **CustomParallel** | **632.97 ms** | **<span style="color: green">-90 %</span>** |

{{< notice success >}}
**<span style="color: green">Le temps est divisé par 2</span>** par rapport à **TinyCsvParser**. Au final, sur mon fichier de 30Go je suis passé de **7min** à **3min20** de traitement de manière simple.
{{< /notice >}}

## Conclusion

Les performances d'une solution personnalisée laissent rêveur, néanmoins ce code ne répond qu'à un seul et unique cas: **les très gros fichiers csv propre et simple**.
Mais dans le cas où vous ayez un grand nombre de fichier csv avec des "qualités" variables, il est fortement recommandé de passer par un framework tel que **CsvHelper** ou **TinyCsvParser** où un grand nombre de bons développeurs ont pu analyser les performances de chaque ligne de code permettant de gérer tout les cas.

{{< alert info >}}
_NB: Il est intéressant de se rendre compte que le classement est totalement chamboulé sur des petits fichiers csv (exemple avec un csv de 10 lignes):_
{{< /alert >}}

| Method          |         Mean |                                           % |
| --------------- | -----------: | ------------------------------------------: |
| TextFieldParser |    253.18 us |                                          0% |
| CsvHelper       |    991.40 us |      <span style="color: red">+291 %</span> |
| TinyCsvParser   |    653.78 us |      <span style="color: red">+158 %</span> |
| **Custom**      | **50.99 us** | **<span style="color: green">-79 %</span>** |
| CustomParallel  |    113.25 us |     <span style="color: green">-55 %</span> |

{{< notice info >}}
_Ce qui prouve encore qu'il n'y a pas de magie en développement, que tout dépend du contexte et annexement que le parallélisme est très souvent profitable et recommandé mais [peut aussi être un piège.](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/potential-pitfalls-in-data-and-task-parallelism)_
{{< /notice >}}

## Sources

### Documentation

- [https://docs.microsoft.com/en-us/dotnet/api/microsoft.visualbasic.fileio.textfieldparser](https://docs.microsoft.com/en-us/dotnet/api/microsoft.visualbasic.fileio.textfieldparser)
- [https://github.com/JoshClose/CsvHelper](https://github.com/JoshClose/CsvHelper)
- [https://github.com/bytefish/TinyCsvParser](https://github.com/bytefish/TinyCsvParser)
- [https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.ienumerable-1.getenumerator](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.ienumerable-1.getenumerator)
- [https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/potential-pitfalls-in-data-and-task-parallelism](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/potential-pitfalls-in-data-and-task-parallelism)
