# EzSubs

## GUI

### Frameworks

Gtk e WX sono supportati abbastanza bene in Haskell, mentre QT lascia piu\` a desiderare.

Pero\` me piacerebbe imparare QT:

* multipiattaforma
* design elegante
* completa e potente
* gira anche su Android alla bisogna

Pragmaticamente riconosco i meriti di WX e sarebbe la mia seconda scelta:

* Audacity e\` scritto con WX
* per il 90% delle applicazioni, quello che WX offre e\` sufficiente
* molto pragmatico, dato che fa principalmente da proxy verso i widget della piattaforma nativa, e quindi alla fine uno richiama la libreria grafica della piattaforma nativa

### Biased Choice

A pelle mi ispira scrivere l'applicazione con QTQuick, collegata con il modello dati in Haskell, usando il package HsQML che permette questo. Eventualmente posso migliorare la comunicazione fra i due sistemi. QTQuick e\` un linguaggio adatto per scrivere GUI:

* struttura a tree dei widget che modella bene il rapporto container -> child widget di una GUI
* la specifica della GUI verrebbe leggibbile e facilmente mantenibile nel tempo
* ha un concetto di signals come in una FRP in Haskell, sotto forma di formule dichiarative (bindings) associate alle properties dei widgtes, e quindi fa quello che farei in Haskell se usassi un FRP

Francamente penso che per quanto sia evoluta una GUI in Haskell, nel migliore dei casi diventerebbe chiara come QTQuick e quindi tanto vale usare l'originale che imitare il tutto in Haskell. L'idea e\` di pensare che QTQuick sia un Domain Specific Language & Run Time per la specifica di GUI in Haskell. In altre parole, non credo che una GUI specificata in FRP + Haskell sia piu\` chiara della GUI in QTQuick, eccetto casi particolari. 

Inoltre mi piace l'idea di una applicazione in cui l'utente puo\` leggere il codice della GUI e arraggianrla in modo diverso, senza dover essere un programmatore esperto. Infatti basta modificare il codice QTQuick.

Per applicazioni commerciali, il fatto di avere del codice QTQuick leggibile, invece che codice compilato, puo\` essere un ostacolo. Ma non e\` il nostro caso e comunque anche dal codice compilato e\` possibile estrarre la sequenza di comandi (spesso ripetitiva) per creare la GUI e sintetizzare una descrizione dichiarativa. Quindi di fatto se uno vuole qualcosa di piu\` criptico puo\` inserire in una stringa compressa/criptata la sequenza di comandi QTQuick e offuscare un po' il tutto. 

### Confirmation of Biased Choice

Provo a studiarmi il mondo delle GUI in Haskell per vedere se la mia scelta sia giusta o meno.

Allora questo sarebbe un esempio di codice QTQuick

    import QtQuick 2.0

    Column {
        height: 300;
        TextInput {
            id: input; width: 300; height: 30; font.pixelSize: 30; focus: true;
        }
        Rectangle {
            color: "red"; width: childrenRect.width; height: childrenRect.height;
            Text {
                width: 300; height: 30; font.pixelSize: 30;
                text: "Calculate Factorial"; color: "white";
            }
            MouseArea {
                anchors.fill: parent;
                onClicked: output.text = factorial(input.text);
            }
        }
        Text {
            width: 300; wrapMode: Text.WrapAnywhere; font.pixelSize: 30;
            id: output;
        }
      }   
    

Lato Haskell bisogna scrivere una sorta di server che risponde ai comandi diretti al Model della GUI e altre cose di questo tipo. Un esempio di codice e\`

    import Graphics.QML
    import Data.Text (Text)
    import qualified Data.Text as T
    
    main :: IO ()
    main = do
        clazz <- newClass [
            defMethod' "factorial" (\_ txt ->
                let n = read $ T.unpack txt :: Integer
                in return . T.pack . show $ product [1..n] :: IO Text)]

Ovviamente man mano che si va avanti nel progetto cerchero\` di capire che tipo di dati il "server" Haskell deve inviare alla GUI QTQuick e cerchero\` di creare un modo compatto lato Haskell per inviare i dati, riducendo il codice boiler-plate al minimo.

Questo e\` un esempio (preso a caso) di GUI in FRP Banana

    {-----------------------------------------------------------------------------
        Main
    ------------------------------------------------------------------------------}
    main :: IO ()
    main = start $ do
        f        <- frame   [ text := "Currency Converter", tabTraversal := True ]
        p        <- panel f []  -- Use panel for tab traversal
        dollar   <- entry p []
        euro     <- entry p []
        
        set p [layout := margin 10 $
                column 10 [
                    grid 10 10 [[label "Dollar:", widget dollar],
                                [label "Euro:"  , widget euro  ]]
                , label "Amounts update while typing."
                ]]
        set f [layout := widget p]
        focusOn dollar
    
        let networkDescription :: MomentIO ()
            networkDescription = do
            
            euroIn   <- behaviorText euro   "0"
            dollarIn <- behaviorText dollar "0"
            
            let rate = 0.7 :: Double
                withString f s
                    = maybe "-" (printf "%.2f") . fmap f 
                    $ listToMaybe [x | (x,"") <- reads s] 
            
                -- define output values in terms of input values
                dollarOut, euroOut :: Behavior String
                dollarOut = withString (/ rate) <$> euroIn
                euroOut   = withString (* rate) <$> dollarIn
        
            sink euro   [text :== euroOut  ]
            sink dollar [text :== dollarOut] 
    
        network <- compile networkDescription    
        actuate network

Definisce gli elementi della GUI come si farebbe in QTQuick, specificando le properties. Definisce dei `behavior` e poi collega i behavior agli elementi della GUI. Usa WX come toolkit grafico.

In questo caso il modello della GUI e\` specificato dentro alla funzione che crea la GUI, ma se fosse esterno, occorre creare (probabilmente) un sistema di comunicazione e mappatura fra model Haskell e GUI simile a quello che e\` necessario usando HsQML.

La GUI in WX, o Gtk e\` piu\` veloce dato che le classi vengono istanziate direttamente senza passare da QTQuick che e\` un simil interprete di Java Script. Di fatto pero\` dopo l'inizializzazione iniziale, il run-time e\` piu\` o meno lo stesso, almeno credo. Inoltre anch nel caso di WX e Gtk, Haskell deve convertire i suoi valori interni, in valori esportabili verso un run-time C, su cui girano le librerie. Nel caso di QTQuick, e\` il run-time QT a gestire il grosso degli eventi FRP, mentre nel caso di Reactive Banana, e\` il run-time a eseguire i calcoli FRP e poi invia il risultato finale ai Widget.



