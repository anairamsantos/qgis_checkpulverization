
import processing as p
from processing.tools import dataobjects
import os
import glob
import errno
import shutil
from qgis.core import QgsProject
from PyQt5.QtCore import QVariant
import pandas as pd
import time
#                       DADOS DE ENTRADA                #
start_time = time.time()
#pasta dos arquivos dxf
path = "D:/BKP MARIANA/PyQGis/Aviacao/PPTA/Janeiro_2020/DXF/"
#arquivos DXF
arq=["01110557","01140521","01140740","01140937","01150459","01150546", "01160500", "01170518", "01170558","01180511","01180530"]
#pasta onde será salvo o shp
shp_path="D:/BKP MARIANA/PyQGis/Aviacao/PPTA/Janeiro_2020/SHP/"
#nome do arquivo de saida
arq_saida="Aviacao_Janeiro_2020"
#pasta onde se encontra abase dos talhoes
path_base="D:/BKP MARIANA/PyQGis/Aviacao/BASE/"
#shp dos talhoes
nome_base="BASE GERAL_31102019.shp"
base_talhoes=path_base+nome_base #caminho+nome da base com os talhoes
#pasta onde está a planilha
planilha_path="D:/BKP MARIANA/PyQGis/Aviacao/PPTA/Janeiro_2020/"
#...NUMEROS DEVEM SER SEPARADOS POR VÍRGULA, CADA TALHAO DEVE ESTAR EM UMA LINHA, SEM ACENTOS
planilha="Janeiro_2020.csv"

#                       PROCESSAMENTO                      
plan = pd.read_csv(planilha_path+planilha)

cod_op=(plan.loc[:,"Cod. Op"])
op=(plan.loc[:,"Operacao"])
secao=(plan.loc[:,"Secao"])
fazenda=(plan.loc[:,"Fazenda"])
nome_setor=(plan.loc[:,"Setor"])
setor=(plan.loc[:,"Set"])
talhao=(plan.loc[:,"Talhao"])
data=(plan.loc[:,"Data"])
hectare=(plan.loc[:,"Area"])

try:
    os.mkdir(shp_path) #cria um diretório onde serão salvos os arquivos transformados (dxf2shp)
except OSError as exc:
    if exc.errno != errno.EEXIST:
        raise OSError("Não é possível criar o diretório "+shp_path)
try:
    os.mkdir (shp_path+'temp/')
except OSError as exc:
    if exc.errno != errno.EEXIST:
        raise OSError('Não foi possível criar o diretório '+shp_path+'temp/')

p.run("qgis:fieldcalculator",{'INPUT':base_talhoes,'FIELD_NAME':'AREA_TALHA','FIELD_TYPE':0,'FIELD_LENGTH':9,'FIELD_PRECISION':2,'NEW_FIELD':True,'FORMULA':'$area/10000','OUTPUT':shp_path+nome_base})
base_talhao=QgsVectorLayer(shp_path+nome_base,"Talhões","ogr")
layervis=QgsProject.instance().addMapLayer(base_talhao)
symbols = layervis.renderer().symbols(QgsRenderContext())
sym = symbols[0]
sym.setColor(QColor.fromRgb(255,255,255))
layervis.triggerRepaint()

i=0 #contador para percorrer os arquivos dxf
while i<len(arq):
    for layer in glob.glob(path + arq[i]+".dxf"): #le o arquivo dxf
        vlayer = QgsVectorLayer(layer, 'name', 'ogr')
        subLayers = vlayer.dataProvider().subLayers()
        for subLayer in subLayers:
            geom_type = subLayer.split(':')[-1]
            uri = "%s|layername=entities|geometrytype=%s" % (layer, geom_type,)
            dfx_file_name = os.path.splitext(os.path.basename(layer))[0]
            layer_name = arq[i]
            sub_vlayer = QgsVectorLayer(uri, layer_name, 'ogr')
            
    #calcula o tamanho das linhas e salva em shp
    p.run("qgis:fieldcalculator",{ 'INPUT':layer,'FIELD_NAME':'length','FIELD_TYPE':0,'FIELD_LENGTH':10,'FIELD_PRECISION':3,'NEW_FIELD':True,'FORMULA':'$length','OUTPUT':shp_path+'temp/'+arq[i]+".shp"}) 
    layer_shp = shp_path+'temp/'+arq[i]+".shp" #le o arquivo shp gerado
    lvis=QgsVectorLayer(layer_shp," ","ogr")
    #layervis=QgsProject.instance().addMapLayer(lvis) #adiciona no QGIS
    maxfeature=lvis.dataProvider().fieldNameIndex('length') #pega a lista com os valorese ID
    maxid = [feat.id() for feat in lvis.getFeatures() if feat['length'] == lvis.maximumValue(maxfeature)] #pega o maior valor/id da coluna length
    with edit(lvis):
        lvis.deleteFeatures(maxid) #deleta a row com maior valor de compriment    i=i+1
        lvis.dataProvider().deleteAttributes([0,1,2,3,4,5,6])
        lvis.updateFields()
    p.run("qgis:linestopolygons", {'INPUT':shp_path+'temp/'+arq[i]+".shp",'OUTPUT':shp_path+'temp/'+arq[i]+"_polygon.shp"})
    i=i+1

i=0
processing.run("native:union", {'INPUT':shp_path+'temp/'+arq[i]+"_polygon.shp",'OVERLAY':shp_path+'temp/'+arq[i+1]+"_polygon.shp",'OUTPUT':shp_path+'temp/'+"intsaida"+str(i)+".shp"})   
while i<(len(arq)-2):
    processing.run("native:union", {'INPUT':shp_path+'temp/'+"intsaida"+str(i)+".shp",'OVERLAY':shp_path+'temp/'+arq[i+2]+"_polygon.shp",'OUTPUT':shp_path+'temp/'+"intsaida"+str(i+1)+".shp"})
    i=i+1
    
p.run("native:intersection", {'INPUT':shp_path+'temp/'+"intsaida"+str(i)+".shp",'OVERLAY':base_talhao,'INPUT_FIELDS':[],'OVERLAY_FIELDS':[],'OUTPUT':shp_path+'temp/'+arq_saida+'1'+'.shp'})

#Dividindo a chave 
UNIDADE =shp_path+'temp/'+'UNIDADE.shp'
SECAO = shp_path+'temp/'+'SECAO.shp'
SETOR = shp_path+'temp/'+'SETOR.shp'

div_UNIDADE= 'substr(LabelText, 1,1)'
div_SECAO='substr(LabelText, 2,4)'
div_SETOR= 'substr(LabelText, 6,4)'
div_TALHAO= 'substr(LabelText, 10,3)'

context = dataobjects.createContext()
context.setInvalidGeometryCheck(QgsFeatureRequest.GeometryNoCheck)
p.run("qgis:fieldcalculator", {'INPUT':shp_path+'temp/'+arq_saida+'1'+'.shp','FIELD_NAME':'UNIDADE','FIELD_TYPE':2,'FIELD_LENGTH':9,'FIELD_PRECISION':3,'NEW_FIELD':True,'FORMULA':div_UNIDADE,'OUTPUT':UNIDADE}, context=context)
p.run("qgis:fieldcalculator", {'INPUT':UNIDADE,'FIELD_NAME':'SECAO','FIELD_TYPE':2,'FIELD_LENGTH':9,'FIELD_PRECISION':3,'NEW_FIELD':True,'FORMULA':div_SECAO,'OUTPUT':SECAO}, context=context)
p.run("qgis:fieldcalculator", {'INPUT':SECAO,'FIELD_NAME':'SETOR','FIELD_TYPE':2,'FIELD_LENGTH':9,'FIELD_PRECISION':3,'NEW_FIELD':True,'FORMULA':div_SETOR,'OUTPUT':SETOR}, context=context)
p.run("qgis:fieldcalculator", {'INPUT':SETOR,'FIELD_NAME':'TALHAO','FIELD_TYPE':2,'FIELD_LENGTH':9,'FIELD_PRECISION':3,'NEW_FIELD':True,'FORMULA':div_TALHAO,'OUTPUT':shp_path+'temp/'+arq_saida+'2'+'.shp'}, context=context)
#p.run("native:dissolve", {'INPUT':shp_path+'temp/'+arq_saida+'2'+'.shp','FIELD':['LabelText'],'OUTPUT':shp_path+'temp/'+arq_saida+'2_setor.shp'})
processing.run("native:dissolve", {'INPUT':shp_path+'temp/'+arq_saida+'2'+'.shp','FIELD':['LabelText'],'OUTPUT':shp_path+'temp/'+arq_saida+'2_setor.shp'}, context=context)
p.run("qgis:fieldcalculator",{'INPUT':shp_path+'temp/'+arq_saida+'2_setor.shp','FIELD_NAME':'AREA_APLIC','FIELD_TYPE':0,'FIELD_LENGTH':9,'FIELD_PRECISION':2,'NEW_FIELD':True,'FORMULA':'$area/10000','OUTPUT':shp_path+'temp/area2_'+arq_saida+'.shp'}, context=context)
p.run("qgis:fieldcalculator", {'INPUT':shp_path+'temp/area2_'+arq_saida+'.shp','FIELD_NAME':'POR_APLIC','FIELD_TYPE':0,'FIELD_LENGTH':9,'FIELD_PRECISION':2,'NEW_FIELD':True,'FORMULA':'AREA_APLIC/AREA_TALHA','OUTPUT':shp_path+arq_saida+'.shp'}, context=context)

layershp_poly=shp_path+arq_saida+'.shp'
lvis_poly=QgsVectorLayer(layershp_poly," ","ogr")
with edit(lvis_poly):
    lvis_poly.dataProvider().deleteAttributes([0,1,2,3,4,5,6])
    lvis_poly.updateFields()

layer_shp = shp_path+arq_saida+'.shp' #le o arquivo shp gerado
lvis=QgsVectorLayer(layer_shp,"Aviação","ogr")
layervis=QgsProject.instance().addMapLayer(lvis) #adiciona no QGIS

symbols = layervis.renderer().symbols(QgsRenderContext())
sym = symbols[0]
sym.setColor(QColor.fromRgb(0,255,100))
layervis.triggerRepaint()

layer = iface.activeLayer()
feature = layer.getFeatures()

unidade=[]
setor_shp=[]
secao_shp=[]
talhao_shp=[]
area_shp=[]
areatalhao_shp=[]
porcentagem_shp=[]


for features in feature:
    un = features.attribute('UNIDADE')
    if un == None :
        unidade.append(None)
    if un != None:
        unidade.append(int(un))
        
    sc = features.attribute('SECAO')
    if sc == None:
        secao_shp.append(None)
    if sc != None:
        secao_shp.append(int(sc))
        
    st = features.attribute('SETOR')
    if st == None:
        setor_shp.append(None)
    if st != None:
        setor_shp.append(int(st))
        
    tl = features.attribute('TALHAO')
    if tl == None:
        talhao_shp.append(None)
    if tl != None:
        talhao_shp.append(int(tl))
        
    area = features.attribute('AREA_APLIC')
    if area == None:
        area_shp.append(None)
    if area != None:
        area_shp.append(float(area))
        
    at = features.attribute('AREA_TALHA')
    if at == None:
        areatalhao_shp.append(None)
    if at != None:
        areatalhao_shp.append(float(at))
    
    pa = features.attribute('POR_APLIC')
    if pa == None:
        porcentagem_shp.append(None)
    if pa != None:
        porcentagem_shp.append(float(pa))
        

saida =open(planilha_path+"Comparacao_"+planilha, 'w')
saida.write("Cod. Op;Operacao;Secao;Fazenda;Set;Setor;Talhao;Data;Area Planilha;Area Total;Area Aplicada;Porcentagem Aplicacao")
saida.write("\n")


j=0
while j<len(setor):
    i=0
    u=0
    while i<len(setor_shp):
        if setor[j] == setor_shp[i] and secao[j] == secao_shp[i] :
            if talhao[j]==talhao_shp[i]:
                saida.write(str(cod_op[j])+";"+(str(op[j])) +";"+(str(secao[j]))+";"+(str(fazenda[j]))+";"+(str(setor[j]))+";"+(str(nome_setor[j]))+";"+(str(talhao[j]))+";"+(str(data[j]))+";"+(str(hectare[j]))+";"+(str(areatalhao_shp[i]))+";"+(str(area_shp[i]))+";"+(str(porcentagem_shp[i]))+("\n"))
                u=u+1
        i=i+1
    if u==0:
        saida.write(str(cod_op[j])+";"+(str(op[j])) +";"+(str(secao[j]))+";"+(str(fazenda[j]))+";"+(str(setor[j]))+";"+(str(nome_setor[j]))+";"+(str(talhao[j]))+";"+(str(data[j]))+";"+(str(hectare[j]))+";"+(str(areatalhao_shp[i-1]))+";"+"Não chegou mapa"+";"+"Não chegou mapa"+("\n"))
    j=j+1

saida.close()

try:
    shutil.rmtree(shp_path+'temp/')
except OSError as exc:
    if exc.errno != errno.EEXIST:
        raise OSError('Não foi possível remover o diretório temp')

stop_time = time.time()
execution_time=(stop_time-start_time)/60
print("O tempo de execucao foi  "+str(execution_time)+ " minutos")
print("PROCESSAMENTO FINALIZADO")