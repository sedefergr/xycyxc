// ==UserScript==
// @name  Travian Kingdoms Animals
// @match https://*.kingdoms.com/*
// ==/UserScript==

(function() {
    'use strict';

    const id = 'animalsButton';
    const modeId = 'animalsModeButton';
    let server = undefined;
    let session = undefined;
    let searching = false;
    let chat = false;

    (function(request) {
        XMLHttpRequest.prototype.send = function() {
            if (typeof arguments[0] === 'string' || arguments[0] instanceof String) {
                if (arguments[0].includes('session')) {
                    session = JSON.parse(arguments[0]).session;
                }
            }

            request.apply(this, arguments);
        };
    })(XMLHttpRequest.prototype.send);

    setInterval(function() {
        if (window.location.href.includes('page:map')) {
            server = window.location.hostname;
            let menu = document.getElementById('userArea');

            if (menu) {
                if (!menu.innerHTML.includes('Zvířata v oázách')) {
                    document.querySelector('#wikiButton').style.display = 'none';
                    document.querySelector('#forumButton').style.display = 'none';

                    const link = document.createElement('a');
                    link.id = id;
                    link.style.width = 'auto';
                    link.style.padding = '0 18px';
                    link.classList.add('headerButton', 'clickable', 'animals');
                    link.innerHTML = '<span class="text" style="display: inline-block; width: 385px;"><span>Zvířata v oázách</span></span><div class="arrow"></div>';
                    menu.appendChild(link);

                    const mode = document.createElement('a');
                    mode.id = modeId;
                    mode.style.width = 'auto';
                    mode.style.padding = '0 18px';
                    mode.classList.add('headerButton', 'clickable', 'animalsMode');
                    mode.innerHTML = `<span class="text" style="display: inline-block; width: 50px;"><span>${chat ? 'Chat' : 'Hra'}</span></span><div class="arrow"></div>`;
                    menu.appendChild(mode);

                    document.querySelector('#animalsModeButton span.text').addEventListener('click', async () => {
                        chat = !chat;

                        document.querySelector('#animalsModeButton span.text span').innerHTML = chat ? 'Chat' : 'Hra';
                    });

                    document.querySelector('#animalsButton span.text').addEventListener('click', async () => {
                        if (searching) {
                            document.querySelector('#animalsButton div.animals').remove();
                            searching = false;

                            return;
                        } else {
                            const div = document.createElement('div');
                            div.classList.add('animals');
                            div.style.width = '385px';
                            div.style.marginTop = '3px';
                            div.style.zIndex = '100';
                            div.style.color = '#000000';
                            const table = document.createElement('table');
                            const cell = table.insertRow().insertCell();
                            cell.style.textAlign = 'center';
                            cell.innerText = 'Hledám zvířata v oázách';
                            div.appendChild(table);
                            link.appendChild(div);

                            searching = true;
                        }

                        const oases = [];
                        let village = undefined;
                        const result = /villId:(\d+)/.exec(window.location.toString());
                        const urlOne = `https://${server}/api/external.php?action=requestApiKey&email=email@example.com&siteName=Example&siteUrl=https://example.com&public=0`;
                        const { response: { privateApiKey } } = await (await fetch(urlOne)).json();
                        const urlTwo = `https://${server}/api/external.php?action=getMapData&privateApiKey=${privateApiKey}`;
                        const { response: { map: { cells }, players } } = await (await fetch(urlTwo)).json();
                        const urlThree = `https://${server}/api/?c=cache&a=get`;
                        const { cache } = await (await fetch(urlThree, {
                            method: 'POST',
                            body: JSON.stringify({
                                controller: 'cache',
                                action: 'get',
                                params: {
                                    names: cells
                                        .filter(({ oasis }) => ['10', '11', '20', '21', '30', '31', '40', '41'].includes(oasis))
                                        .map(({ id, x, y }) => {
                                            oases.push({
                                                id: parseInt(id),
                                                x: parseInt(x),
                                                y: parseInt(y),
                                            });

                                            return `MapDetails:${id}`;
                                        }),
                                    session,
                                },
                            }),
                        })).json();

                        if (result) {
                            players.map(({ villages }) => {
                                villages.map(({ villageId, x, y }) => {
                                    if (villageId === result[1] && !village) {
                                        village = {
                                            x: parseInt(x),
                                            y: parseInt(y),
                                        };
                                    }

                                    return undefined;
                                });

                                return undefined;
                            });
                        }

                        const withAnimals = cache.filter(({ data: { oasisStatus } }) => !['1'].includes(oasisStatus));
                        const table = document.createElement('table');
                        table.style.borderTop = '0';
                        table.style.borderBottom = '0';
                        const text = oases.map(oasis => {
                            const animals = withAnimals.map(({ name, data: { troops: { units } } }) => {
                                if (oasis.id === parseInt(name.split(':')[1])) {
                                    return {
                                        1: 0,
                                        2: 0,
                                        3: 0,
                                        4: 0,
                                        5: 0,
                                        6: 0,
                                        7: 0,
                                        8: 0,
                                        9: 0,
                                        10: 0,
                                        ...Object
                                            .entries(units)
                                            .reduce((accumulator, [key, value]) => ({ ...accumulator, [key]: parseInt(value) }), {}),
                                    };
                                }

                                return undefined;
                            }).filter(value => value !== undefined);

                            return animals.length === 1 && (typeof animals[0] === 'object' || animals[0] instanceof Object) ?
                                { ...oasis, animals: animals[0] } :
                                undefined;
                        })
                            .filter(v => v !== undefined)
                            .sort(({ animals: animalsOne }, { animals: animalsTwo }) => {
                                let sortation = undefined;

                                [10, 9, 8, 7, 6, 5, 4, 3, 2, 1].map(animal => {
                                    if (animalsOne[animal] !== animalsTwo[animal] && !sortation) {
                                        sortation = animalsOne[animal] > animalsTwo[animal] ? -1 : 1;
                                    }
                                });

                                return sortation ? sortation : animalsOne[1] > animalsTwo[1] ? -1 : 1;
                            })
                            .map(({ id, x, y, animals }) => {
                                const style = 'display: inline-block; width: 20px; height: 20px; margin: 0 auto; vertical-align: -4px; background-image: url("https://cdn.traviantools.net/game/0.94/layout/images/sprites/unit/small/unit/small.png"); background-position: ';
                                const content = [];
                                const convert = {
                                    10: { name: 'Elephant', position: '-40px -80px' },
                                    9: { name: 'Tiger', position: '-20px -100px' },
                                    8: { name: 'Crocodile', position: '0 -100px' },
                                    7: { name: 'Bear', position: '-100px -80px' },
                                    6: { name: 'Wolf', position: '-100px -60px' },
                                    5: { name: 'Boar', position: '-100px -40px' },
                                    4: { name: 'Bat', position: '0 0' },
                                    3: { name: 'Snake', position: '-100px 0' },
                                    2: { name: 'Spider', position: '-80px -80px' },
                                    1: { name: 'Rat', position: '-60px -80px' },
                                };

                                Object
                                    .entries(convert)
                                    .reverse()
                                    .map(([animal, { name, position }]) => {
                                        if (animals[animal]) {
                                            content.push(
                                                chat ?
                                                    `${animals[animal]} ${name}` :
                                                    `${animals[animal].toString().padStart(3, '0')} <div style='${style} ${position};'></div>`,
                                            );
                                        }

                                        return undefined;
                                    });

                                if (content.length) {
                                    const row = table.insertRow();
                                    const distance = village ?
                                        ` vzdálená ${Math.sqrt(Math.pow(x - village.x, 2) + Math.pow(y - village.y, 2))
                                            .toFixed(0)
                                            .toString()
                                            .padStart(3, '0')} polí` :
                                        '';
                                    row.insertCell().innerHTML = chat ? `[village:${id}]` : `<a href="https://${server}/#/page:map/centerId:${id}/cellId:${id}">Oáza</a>${distance}`;
                                    row.insertCell().innerHTML = content.join(chat ? ', ' : '&nbsp;');

                                    return undefined;
                                }

                                return undefined;
                            }).filter(v => v !== undefined);

                        document.querySelector('#animalsButton div.animals').remove();
                        const div = document.createElement('div');
                        div.classList.add('animals');
                        div.style.display = 'block';
                        div.style.width = '385px';
                        div.style.height = '170px';
                        div.style.marginTop = '3px';
                        div.style.overflowY = 'scroll';
                        div.style.zIndex = '100';
                        div.style.color = '#000000';
                        div.appendChild(table);
                        link.appendChild(div);
                    });
                } else {
                    let link = document.getElementById(id);
                    let mode = document.getElementById(modeId);

                    if (link && mode) {
                        link.style.display = 'inherit';
                        mode.style.display = 'inherit';
                        document.querySelector('#wikiButton').style.display = 'none';
                        document.querySelector('#forumButton').style.display = 'none';
                    }

                }
            }
        } else {
            let link = document.getElementById(id);
            let mode = document.getElementById(modeId);

            if (link && mode) {
                link.style.display = 'none';
                mode.style.display = 'none';
                document.querySelector('#wikiButton').style.display = 'inherit';
                document.querySelector('#forumButton').style.display = 'inherit';
            }
        }
    }, 1000);

})();
