---
date: 2023-12-26
slug: lendo-arquivos-pdf-com-apache-pdfbox-3
readtime: 15
authors:
  - vmutter
categories:
  - Java
  - Apache
tags:
  - Java
  - PDF
  - Apache
  - Apache PDFBox 3
comments: true
---

# Lendo arquvios PDF com Java usando o Apache PDFBox 3

A manipulação de arquivos PDF é uma tarefa comum em muitos projetos de software. Se você está trabalhando com Java, uma das melhores bibliotecas disponíveis para esta finalidade é o Apache PDFBox 3.

## Introdução ao Apache PDFBox 3

O Apache PDFBox é uma biblioteca open-source que permite a criação, manipulação e obtenção de conteúdo de arquivos PDF. A versão 3 do Apache PDFBox trouxe melhorias significativas em termos de performance e facilidade de uso.

Primeiro precisamos adicionar a biblioteca do Apache PDFBox no projeto, no `pom.xml` adicionar:

``` xml title="pom.xml"
<dependency>
    <groupId>org.apache.pdfbox</groupId>
    <artifactId>pdfbox</artifactId>
    <version>3.0.0</version>
</dependency>
```

<!-- more -->

## Extração de Texto de um Documento Inteiro

Vamos começar com um exemplo simples de como extrair texto de um documento PDF inteiro.

``` java title="ExtractTextFromPDF.java"
import org.apache.pdfbox.pdmodel.PDDocument;
import org.apache.pdfbox.text.PDFTextStripper;

public class ExtractTextFromPDF {
    public static void main(String[] args) {
        // Carregando o arquivo "seu_documento.pdf" da pasta resources
        ClassLoader classLoader = getClass().getClassLoader();
        File file = new File(classLoader.getResource("seu_documento.pdf").getFile());

        try (PDDocument document = PDDocument.load(file)) {
            PDFTextStripper stripper = new PDFTextStripper();
            String text = stripper.getText(document);
            System.out.println("Texto extraído: \n" + text);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

Neste exemplo, carregamos o documento PDF usando PDDocument.load e utilizamos a classe PDFTextStripper para extrair o texto. O método getText retorna o texto extraído, que é então impresso no console.

## Extração de Texto por Área

A extração de texto por área é útil quando você está interessado apenas em uma parte específica do documento.

``` java title="ExtractTextByArea.java"
import org.apache.pdfbox.pdmodel.PDDocument;
import org.apache.pdfbox.text.PDFTextStripperByArea;

import java.awt.Rectangle;
import java.awt.geom.Rectangle2D;

public class ExtractTextByArea {
    public static void main(String[] args) {
        // Carregando o arquivo "seu_documento.pdf" da pasta resources
        ClassLoader classLoader = getClass().getClassLoader();
        File file = new File(classLoader.getResource("seu_documento.pdf").getFile());

        try (PDDocument document = PDDocument.load(file)) {
            PDFTextStripperByArea stripper = new PDFTextStripperByArea();
            stripper.setSortByPosition(true);

            Rectangle2D region = new Rectangle(10, 50, 200, 100); // Define a área (x, y, largura, altura)
            stripper.addRegion("region1", region);

            stripper.extractRegions(document.getPage(0)); // Extrai texto da primeira página
            String textForRegion = stripper.getTextForRegion("region1");
            System.out.println("Texto extraído da área especificada: \n" + textForRegion);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

Neste exemplo, definimos uma região específica usando um objeto Rectangle2D e adicionamos essa região ao PDFTextStripperByArea. Depois, extraímos o texto dessa área específica e o imprimimos no console.

## Conclusão

O Apache PDFBox 3 oferece um meio eficaz e flexível de trabalhar com arquivos PDF em Java. Seja para extrair texto de um documento inteiro ou de áreas específicas, essa biblioteca se mostra bastante útil em diversas situações. Com a facilidade de uso e a robustez do PDFBox, você pode integrar facilmente a manipulação de PDFs em seus projetos Java.
