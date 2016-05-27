# reproductor_musica
/*EN ESTE TRABAJO UTILIZE ALGUNAS BASES PARA PODER REALIZARLO, NO FUERON CONOCIMIENTOS MIOS PROPIOS QUE PLASME, USE UN POCO DE EJEMPLOS, TUTORIALES Y COSAS 
POR EL ESTILO NO ME QUERIA SALIR EL ECUALIZADOR ESE ME LO EXPLICO UN COMPAÑERO YA QUE AL MOMENTO DE USARLO EL PROGRAMA SE TRONABA, EL PROBLEMA ESTABA EN QUE LO TENIA DONDE NO DEBIA
Y AL MOMENTO DE MANDARLO LLAMAR PARA QUE MODIFICARA LA CANSION SE INTERFERIA ENTONCES BUUUM NO FUNCIONABA Y ME TRONABA EL PROGRAMA, COMENTE PARTE DEL CODIGO UNAS YA VENIAN COMENTADAS
YA QUE SON DE LA BASE DE DATOS. ALGUNAS COSAS QUE ENCONTRE EN INERNET NI LAS MODIFIQUE SOLO COPEAR, PEGAR Y HACER QUE FUNCIONE YA QUE TENIA MAS TAREAS PROYECTOS FINALES Y EXAMENES
DE LA SEMANA Y CASI NO E DORMIDO EN DIAS. MUERO DE SUEÑO Y A UN NO TERMINO Y YA CASI SON LAS 3 DE LA MAÑANA :v */
import ddf.minim.*;
import controlP5.*;
import ddf.minim.analysis.*;
import ddf.minim.effects.*;
import ddf.minim.ugens.*;

import controlP5.*;
import ddf.minim.*;

import java.util.*;
import java.net.InetAddress;
import javax.swing.*;
import javax.swing.filechooser.FileFilter;
import javax.swing.filechooser.FileNameExtensionFilter;

import org.elasticsearch.action.admin.indices.exists.indices.IndicesExistsResponse;
import org.elasticsearch.action.admin.cluster.health.ClusterHealthResponse;
import org.elasticsearch.action.index.IndexRequest;
import org.elasticsearch.action.index.IndexResponse;
import org.elasticsearch.action.search.SearchResponse;
import org.elasticsearch.action.search.SearchType;
import org.elasticsearch.client.Client;
import org.elasticsearch.common.settings.Settings;
import org.elasticsearch.node.Node;
import org.elasticsearch.node.NodeBuilder;

ControlP5 play,detener,pause,slid,cargar;
HighPassSP highpass;
LowPassSP lowpass;
BandPass bandpass;
Button btnPlay,btnPause;
LowPassFS   lpf;
Minim min;
AudioPlayer musica;
boolean variable, m=false;
float NivelVolumen,Auxiliar;
AudioMetaData datos;
PImage playy,stopp,pausee,cargaa;
Textlabel texto;
FFT          fft;

int Hpass;
int Lpass;
int Bpass;

static String INDEX_NAME = "canciones";
static String DOC_TYPE = "cancion";

ControlP5 cp5;
ScrollableList list;

Client client;
Node node;

void setup(){
 size(500,300);
 background(255);
 cp5 = new ControlP5(this);

  // Configuracion basica para ElasticSearch en local
  Settings.Builder settings = Settings.settingsBuilder();
  // Esta carpeta se encontrara dentro de la carpeta del Processing
  settings.put("path.data", "esdata");
  settings.put("path.home", "/");
  settings.put("http.enabled", false);
  settings.put("index.number_of_replicas", 0);
  settings.put("index.number_of_shards", 1);

  // Inicializacion del nodo de ElasticSearch
  node = NodeBuilder.nodeBuilder()
          .settings(settings)
          .clusterName("mycluster")
          .data(true)
          .local(true)
          .node();

  // Instancia de cliente de conexion al nodo de ElasticSearch
  client = node.client();

  // Esperamos a que el nodo este correctamente inicializado
  ClusterHealthResponse r = client.admin().cluster().prepareHealth().setWaitForGreenStatus().get();
  println(r);

  // Revisamos que nuestro indice (base de datos) exista
  IndicesExistsResponse ier = client.admin().indices().prepareExists(INDEX_NAME).get();
  if(!ier.isExists()) {
    // En caso contrario, se crea el indice
    client.admin().indices().prepareCreate(INDEX_NAME).get();
  }

  // Agregamos a la vista un boton de importacion de archivos
  cp5.addButton("importFiles")
    .setPosition(310, 10)
    .setLabel("Importar archivos");

  // Agregamos a la vista una lista scrollable que mostrara las canciones
  list = cp5.addScrollableList("playlist")
            .setPosition(310, 40)
            .setSize(200, 400)
            .setBarHeight(20)
            .setItemHeight(20)
            .setType(ScrollableList.LIST);

  // Cargamos los archivos de la base de datos
  loadFiles();
 playy = loadImage("Play_blauw.png");
 stopp = loadImage("Crystal_Project_Player_stop.png");
 pausee = loadImage("Button Pause.png");
 play = new ControlP5(this);
 playy.resize(30,30);
 btnPlay = play.addButton("play").setValue(0).setPosition(0,270).setSize(30,30).setImage(playy);
 texto = play.addTextlabel("label")
                    .setText("Titulo Desconocido \nInterprete Desconocido")
                    .setPosition(0,0)
                    .setColorValue(255)
                    .setFont(createFont("Century Gothic",10))
                    ;
 Auxiliar= NivelVolumen;
 pause = new ControlP5(this);
 pausee.resize(30,30);
 btnPause = pause.addButton("pause").setValue(0).setPosition(0,270).setSize(30,30).setImage(pausee).hide();
 
 detener = new ControlP5(this);
 stopp.resize(30,30);
 detener.addButton("stop").setValue(0).setPosition(30,270).setSize(40,40).setImage(stopp);
 
//EN ESTA PARTE ES DONDE DEFINIMOS DONDE ESTARA COLOCADO, RANGO Y CARACTERISTICAS DE LAS 3 BARRAS PARA MODIFICAR EL AUDIO QUE SON 3
 slid= new ControlP5(this);
 slid.setColorForeground(255);
  slid.setColorBackground(0);
  slid.setColorActive(0xffff0000);
  cp5.addSlider("Hpass")
     .setPosition(250,100)
     .setSize(10,100)
     .setRange(0,3000)
     .setValue(0)
     .setNumberOfTickMarks(30);
     
 cp5.addSlider("Bpass")
     .setPosition(50,100)
     .setSize(10,100)
     .setRange(100,1000)//250-2500
     .setValue(100)
     .setNumberOfTickMarks(30);
     
      cp5.addSlider("Lpass")
     .setPosition(150,100)
     .setSize(10,100)
     .setRange(3000,20000) //100-150
     .setValue(3000)
     .setNumberOfTickMarks(30);
     
     
     
 cp5.getController("Hpass").getValueLabel().align(ControlP5.RIGHT, ControlP5.BOTTOM_OUTSIDE).setPaddingY(-100);
 cp5.getController("Lpass").getValueLabel().align(ControlP5.RIGHT, ControlP5.BOTTOM_OUTSIDE).setPaddingY(-100);
 cp5.getController("Bpass").getValueLabel().align(ControlP5.RIGHT, ControlP5.BOTTOM_OUTSIDE).setPaddingY(-100);
 slid.addSlider("Volumen").setValue(50).setPosition(150,280).setSize(100,10);
 min = new Minim(this);}
 


void importFiles() {
  // Selector de archivos
  JFileChooser jfc = new JFileChooser();
  // Agregamos filtro para seleccionar solo archivos .mp3
  jfc.setFileFilter(new FileNameExtensionFilter("MP3 File", "mp3"));
  // Se permite seleccionar multiples archivos a la vez
  jfc.setMultiSelectionEnabled(true);
  // Abre el dialogo de seleccion
  jfc.showOpenDialog(null);

  // Iteramos los archivos seleccionados
  for(File f : jfc.getSelectedFiles()) {
    // Si el archivo ya existe en el indice, se ignora
    GetResponse response = client.prepareGet(INDEX_NAME, DOC_TYPE, f.getAbsolutePath()).setRefresh(true).execute().actionGet();
    if(response.isExists()) {
      continue;
    }

    // Cargamos el archivo en la libreria minim para extrar los metadatos
    Minim minim = new Minim(this);
    AudioPlayer song = minim.loadFile(f.getAbsolutePath());
    AudioMetaData meta = song.getMetaData();

    // Almacenamos los metadatos en un hashmap
    Map<String, Object> doc = new HashMap<String, Object>();
    doc.put("author", meta.author());
    doc.put("title", meta.title());
    doc.put("path", f.getAbsolutePath());

    try {
      // Le decimos a ElasticSearch que guarde e indexe el objeto
      client.prepareIndex(INDEX_NAME, DOC_TYPE, f.getAbsolutePath())
        .setSource(doc)
        .execute()
        .actionGet();

      // Agregamos el archivo a la lista
      addItem(doc);
    } catch(Exception e) {
      e.printStackTrace();
    }
  }
}

// Al hacer click en algun elemento de la lista, se ejecuta este metodo
void playlist(int n) {
   //println(list.getItem(n));
   if(musica!=null){musica.pause();}
   Map<String, Object> value = (Map<String, Object>) list.getItem(n).get("value");
   println(value.get("path"));
   min = new Minim(this);
     
 musica = min.loadFile((String)value.get("path"),1024);
 btnPlay.show();
  btnPause.hide();
 highpass = new HighPassSP(300, musica.sampleRate());
   musica.addEffect(highpass);
   lowpass = new LowPassSP(300, musica.sampleRate());
   musica.addEffect(lowpass);
   bandpass = new BandPass(300, 300, musica.sampleRate());
   musica.addEffect(bandpass);
 fft = new FFT(musica.bufferSize(), musica.sampleRate());
  // calculate averages based on a miminum octave width of 22 Hz
  // split each octave into three bands
  fft.logAverages(22, 10);
 datos = musica.getMetaData();
  if(!datos.title().equals("")){texto.setText(datos.title()+"`\n"+datos.author());print("sale");}
  else{texto.setText(datos.fileName());print("entra");}
   //musica = min.loadFile(selection.getAbsolutePath(),1024);
   
}

void loadFiles() {
  try {
    // Buscamos todos los documentos en el indice
    SearchResponse response = client.prepareSearch(INDEX_NAME).execute().actionGet();

    // Se itera los resultados
    for(SearchHit hit : response.getHits().getHits()) {
      // Cada resultado lo agregamos a la lista
      addItem(hit.getSource());
    }
  } catch(Exception e) {
    e.printStackTrace();
  }
}

// Metodo auxiliar para no repetir codigo
void addItem(Map<String, Object> doc) {
  // Se agrega a la lista. El primer argumento es el texto a desplegar en la lista, el segundo es el objeto que queremos que almacene
  list.addItem(doc.get("author") + " - " + doc.get("title"), doc);
}
void draw(){
  
  //ESTO NO TIENE MUCHO QUE VER PERO PUSE DOS RECTAS QUE ME AYDUAN A DIVIDIARI EL "MENU" DE LA BSE DE DATOS CON EL ECUALIZADOR Y A SI
  background(150,255);
  fill(0);
  
  rect(0,0,290,270);
  
  rect(300,0,500,270);
   
  if(musica!=null){
    highpass.setFreq(Hpass);
    lowpass.setFreq(Lpass);
    bandpass.setFreq(Bpass);
   fill(random(255), random(0), random(0));
   fft.forward(musica.mix);
   int w = int((width-200)/fft.avgSize());
  for(int i = 0; i < fft.avgSize(); i++)
  {
    // Dibujar un rectángulo para cada medio, multiplicar el valor por lo que podemos ver mejor
   rect(i*w, height-30, i*w + w,height-30 - fft.getAvg(i)*3);
  }
  }
}

//AQUI COLOQUE LAS FUNCIONES DE LOS BOTONES QUE TENGO PAUSA STOP Y PLAY
public void play(){
  musica.play();
  btnPlay.hide();
  btnPause.show();
}

public void controlEvent(ControlEvent event){
  
}

public void stop(){
  musica.pause();
  musica.rewind();
}

public void detener(){
stop();
}

public void pause(){
  musica.pause();
  btnPlay.show();
  btnPause.hide();
}

//CON ESTO LO QUE HAGO ES AUMENTAR O BJAR EL VOLUMEN DE LA CANSOON QUE TENGO 
public void Volumen(float volumen){
  float Volumen=volumen-25;
  
 if(Volumen>50){
 musica.setGain(Volumen);
  }else{
  musica.setGain(Volumen);
 }
}

//PARA PODER SELECCIONAR MI ARCHIVO A PONERLO A REPRODUCIR 
void fileSelected(File selection) {
  if (selection == null) {
    println("Window was closed or the user hit cancel.");
  } else {
    if(musica!=null){musica.pause();}
    println("User selected " + selection.getAbsolutePath());
     min = new Minim(this);
     
     //ESTO ES LO QUE HACE POSIBLE MODIFICAR EL AUDIO POR HIGHT, LOW Y BAND SON 3 EJEMPLOS QUE ENCONTRE 
 musica = min.loadFile(selection.getAbsolutePath(),1024);
 highpass = new HighPassSP(300, musica.sampleRate());
   musica.addEffect(highpass);
   lowpass = new LowPassSP(300, musica.sampleRate());
   musica.addEffect(lowpass);
   bandpass = new BandPass(300, 300, musica.sampleRate());
   musica.addEffect(bandpass);
 fft = new FFT(musica.bufferSize(), musica.sampleRate());
 
  fft.logAverages(22, 10);
 datos = musica.getMetaData();
  if(!datos.title().equals("")){texto.setText(datos.title()+"`\n"+datos.author());print("sale");}
  else{texto.setText(datos.fileName());print("entra");}
 
 btnPlay.show();
  btnPause.hide();
  }
}
