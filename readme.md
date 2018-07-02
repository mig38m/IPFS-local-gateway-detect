Переключаем наш сайт

DNS

У нашего сайта на данный момент уже есть как минимум 3 DNS записи:


A    [Наш домен]             [IP адрес хостинга]
TXT  [Наш домен]             dnslink=/ipfs/[CID контента]
TXT  _dnslink.[Наш домен]    dnslink=/ipfs/[CID контента]

Добавим к ним ещё 3:


A    l.[Наш домен]             127.0.0.1
TXT  l.[Наш домен]             dnslink=/ipfs/[CID контента]
TXT  _dnslink.l.[Наш домен]    dnslink=/ipfs/[CID контента]

[CID контента] — Это идентификатор контента (CID) раньше назывался мультихеш. Его мы получаем публикуя сайт командой ipfs add в сети IPFS.


Скрипты и стили

В HTML тегах script и link появились поля integrity и crossorigin. Они отвечают за проверку хеша до запуска скрипта или применения стилей. Их мы и используем для определения рабочего шлюза у посетителя сайта.


<script src="[адрес скрипта для проверки]" 
        integrity="sha384-[sha384 в base64]"
        crossorigin="anonymous" defer async>
</script>
<link rel="stylesheet" href="[адрес стиля для проверки]"
      integrity="sha384-[sha384 в base64]"
      crossorigin="anonymous"/>

Расположить их лучше в конце страницы, чтобы они не задерживали загрузку и отображение.


Варианты адресов, которые нам надо проверить:


http://l.[наш домен]:8080
8080 это стандартный порт на котором по умолчанию запускается IPFS.
Если всё настроено правильно, то с http-версии сайта браузер загрузит скрипт или стиль.


https://l.[наш домен]:8443
8443 это порт на который пользователь может настроить stunnel.
Данный вариант нам понадобится, если запрос идёт с HTTPS сайта и наш домен добавлен в локальный сертификат.


http://127.0.0.1:8080/ipns/[наш домен]
Этот вариант на случай, если мы не задали l домен для нашего сайта и запрос идёт с http.


https://127.0.0.1:8443/ipns/[наш домен]
Этот вариант на случай запроса с https. Этот вариант сработает, если не задан l домен или не добавлен в локальный сертификат.

Аналогичным образом мы можем проверить порты 80 для http и 443 для https.


Проверяем локальный шлюз и переключаемся на него скриптом

Добавив этот скрипт к странице вашего сайта вы автоматически переключите посетителя на его локальный IPFS шлюз.


<script src="redirect_to_ipfs_gateway.js" defer async ></script>

redirect_to_ipfs_gateway.js


var redirect_to_local;

/*
Эта функция добавляет к текущему домену третьим уровнем  домен ```l``` 
*/

function l_hostname()
{
    var l_hostname =  window.location.hostname.split(".");
    l_hostname.splice(-2,0,"l");
    return l_hostname.join(".");
}

/*
Эта функция создаёт новый элемент script и с адресом скрипта, который должен загрузиться через локальный шлюз пользователя.

Также она создаёт функцию, которая будет вызвана этим скриптом в случае удачной загрузки.

В случае неудачи выполнится функция из переменной onerror, которая присваивается соответствующему полю элемента script.
*/

function add_redirect_script(prtocol, port, use_ip, onerror){
    var script = document.createElement("script");
    script.onerror = onerror;
    script.setAttribute("integrity", "sha384-dActyNwOxNY9fpUWleNW9Nuy3Suv8Dx3F2Tbj1QTZWUslB1h23+xUPillTDxprx7");
    script.setAttribute("crossorigin", "anonymous");
    script.setAttribute("defer", "");
    script.setAttribute("async", "");
    if ( use_ip )
        script.setAttribute("src", prtocol+"//127.0.0.1:"+port+"/ipns/"+window.location.hostname+"/redirect_call.js");
    else
        script.setAttribute("src", prtocol+"//"+l_hostname()+":"+port+"/redirect_call.js");

    redirect_to_local = function()
    {
        var a = document.createElement("a");
        a.href = window.location;
        a.protocol = prtocol;
        a.port = port;
        if ( use_ip ){
            a.pathname = "/ipns/" + a.hostname + a.pathname;
            a.hostname = "127.0.0.1";
        }else{
            var hostname = a.hostname.split(".");
            hostname.splice(-2,0,"l");
            a.hostname = hostname.join(".");
        }
        window.location = a.href;
    };
    document.head.appendChild(script);
}

/*
Это главная функция, которая запускается сразу. Она проверяет, не находимся ли мы уже по адресу шлюза. Если нет, то начинает проверять его доступность, перебирая варианты адресов и протоколов.
*/
!function(location){
    if ( location.protocol.indexOf("http") == 0 &&
         location.hostname.length          >  0 &&
         location.hostname.indexOf("l.")   != 0 &&
         location.hostname.indexOf(".l.")  <  0 &&
         location.hostname                 != "127.0.0.1" ) 
    {   
        add_redirect_script( "http:",  8080, false,
         function(){
            add_redirect_script( "https:", 8443, false,
             function(){
                add_redirect_script( "http:", 8080, true, 
                 function(){
                    add_redirect_script( "https:", 8443, true );
                 } );
             } );
         } );
    }
}(window.location)

В пару ему идет скрипт:
redirect_call.js (sha384-dActyNwOxNY9fpUWleNW9Nuy3Suv8Dx3F2Tbj1QTZWUslB1h23+xUPillTDxprx7)


redirect_to_local();

Integrity этого скрипта можно посчитать командой:


openssl dgst -sha384 -binary < "redirect_call.js" | openssl enc -base64 -A

У меня соответственно результат этой команды:


dActyNwOxNY9fpUWleNW9Nuy3Suv8Dx3F2Tbj1QTZWUslB1h23+xUPillTDxprx7

Если у вас результат другой, замените это значение в скрипте выше на своё.


Теперь пользователь будет автоматически перенаправлен на подходящий локальный адрес шлюза с сохранением остальных параметров адреса.


Определяем рабочий шлюз при помощи CSS

Создадим CSS файл который будет маяком работы шлюза.


httpl.css (sha384-4zVfUIU3jAvrXNQlD5WHMl6TI8bMiBseKrDfLJpr5Mn+xdygya+svSeP6dK/5wpu)


.httpl{;display: block;}

Скрываем элементы страницы, которые будут показаны только при доступности локального шлюза.


<style>
.gatewaylink{display: none}
</style>

Добавляем CSS маяк в конце страницы


<link rel="stylesheet" href="http://l.[наш домен]:8080/httpl.css" 
     integrity="sha384-4zVfUIU3jAvrXNQlD5WHMl6TI8bMiBseKrDfLJpr5Mn+xdygya+svSeP6dK/5wpu"
     crossorigin="anonymous" />

Этот элемент будет отображаться если загрузиться "httpl.css"
<div class="gatewaylink httpl">
Use your gateway: <a href="http://l.[наш домен]:8080/">http://l.[наш домен]:8080/</a> 
</div>

Теперь даже если у пользователя отключены скрипты, он сможет перейти на шлюз самостоятельно по ссылке. Аналогично можно проверить и остальные варианты адреса шлюза.


Сайт для теста: ivan386.ml


GitHub:


JavaScript detect local IPFS gateway and switch to it.

Источники:


Subresource Integrity

Локализуем глобальный шлюз или сайты в IPFS

Продолжение перенесено: Локализуем глобальный шлюз или сайты в IPFS


Другие мои статьи о "межпланетной файловой системе":


"Межпланетная файловая система IPFS"
Публикуем сайт в межпланетной файловой системе IPFS
Хостим сайт в межпланетной файловой системе IPFS под Windows
Больше нет необходимости копировать в сеть
Локализуем глобальный шлюз или сайты в IPFS
