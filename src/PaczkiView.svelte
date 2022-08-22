<script>
    import { default as paczki_plusplus_proto } from "./external/paczki_plusplus.js";
    import { default as packet_viewer_proto } from "./external/packet_viewer.js";
    import { onMount } from "svelte";
    import { writable, get } from "svelte/store";
    import Card, { Content } from "@smui/card";
    import LayoutGrid, { Cell as CellLG } from "@smui/layout-grid";
    import Button, { Label } from "@smui/button";
    import Switch from "@smui/switch";
    import Slider from "@smui/slider";
    import FormField from "@smui/form-field";
    import Dialog, {
        Title,
        Content as DialogContent,
        Actions,
    } from "@smui/dialog";

    import { saveAs } from "file-saver";

    import DataTable, { Head, Body, Row, Cell } from "@smui/data-table";

    let paczki_plusplus_canvas;
    let packet_viewer_canvas;

    let paczki_plusplus;
    let packet_viewer;

    const active_packet = writable(null);
    const active_box_type = writable(null);
    const active_sku = writable(null);
    const box_type_list = writable(null);
    const box_pos_list = writable(null);
    const pallet_id_list = writable(null);
    const sku_list = writable(null);
    const selected_pallet = writable(null);
    const box_pos_order = writable([[]]);
    const box_pos_order_it = writable(-1);
    const box_clipboard = writable([{}]);
    const color_by_box_pos = writable(true);

    const stores = {
        active_packet: active_packet,
        active_box_type: active_box_type,
        active_sku: active_sku,
        box_type_list: box_type_list,
        box_pos_list: box_pos_list,
        pallet_id_list: pallet_id_list,
        sku_list: sku_list,
        selected_pallet: selected_pallet,
        box_pos_order: box_pos_order,
        box_pos_order_it: box_pos_order_it,
        box_clipboard: box_clipboard,
        color_by_box_pos: color_by_box_pos,
    };

    let pallet_only_box_types = true;
    $: box_type_val =
        $box_type_list && $active_box_type
            ? $box_type_list[$active_box_type]
            : null;
    $: box_type_list_v = $box_type_list ? Object.values($box_type_list) : [];
    $: pallet_box_type_list = $box_type_list
        ? box_type_list_v.filter((box_type) => {
              return Object.fromEntries($pallet_id_list)[$selected_pallet][
                  box_type.$id
              ];
          })
        : {};
    $: box_types =
        pallet_only_box_types && $box_type_list
            ? pallet_box_type_list
            : $box_type_list
            ? box_type_list_v
            : [];

    $: items_per_box = $box_type_list
        ? Object.fromEntries(
              box_type_list_v.map((box_type) => {
                  let map = {};
                  let items = box_type.Num
                      ? box_type.Items
                      : box_type.converted_sku;
                  for (let item of items) {
                      map[item.Item1] = map[item.Item1]
                          ? map[item.Item1] + 1
                          : 1;
                  }
                  return [box_type["$id"], map];
              })
          )
        : {};

    const setupStore = (module_proper, store_name) => {
        let store_obj = stores[store_name];
        store_obj.subscribe((value) => {
            module_proper.callUpdate(store_name);
        });
        module_proper.addStore(store_name, store_obj);
        window[store_name] = store_obj;
    };

    let value = 0;

    onMount(() => {
        const packet_viewer_stores = [
            "active_packet",
            "active_box_type",
            "active_sku",
            "sku_list",
            "box_type_list",
        ];
        packet_viewer_proto({
            canvas: packet_viewer_canvas,
            onExit: () => {
                showAlert("Aplikacja niespodziewanie zamknięta :(", true);
                window.location.reload();
            },
        }).then((self) => {
            packet_viewer = self;
            window.packet_viewer = packet_viewer;
            packet_viewer.setGet(get);
            packet_viewer_stores.forEach((name) => {
                setupStore(packet_viewer, name);
            });
        });

        const paczki_plusplus_stores = [
            "active_box_type",
            "active_packet",
            "active_sku",
            "sku_list",
            "box_type_list",
            "box_pos_list",
            "pallet_id_list",
            "selected_pallet",
            "box_pos_order",
            "box_clipboard",
            "color_by_box_pos",
            "box_pos_order_it",
        ];
        paczki_plusplus_proto({
            canvas: paczki_plusplus_canvas,
            onExit: () => {
                showAlert("Aplikacja niespodziewanie zamknięta :(", true);
                window.location.reload();
            },
        }).then((self) => {
            paczki_plusplus = self;
            window.paczki_plusplus = paczki_plusplus;
            paczki_plusplus.setGet(get);
            paczki_plusplus_stores.forEach((name) => {
                setupStore(paczki_plusplus, name);
            });
        });
    });

    function offerDownload(filename) {
        let content = paczki_plusplus.FS.readFile(filename);
        console.log(
            `Offering download of "${filename}", with ${content.length} bytes...`
        );
        saveAs(new Blob([content], { type: "application/json" }), filename);
    }
    window.offerDownload = offerDownload;

    function openFile() {
        var passToCpp = function (e) {
            const file_reader = new FileReader();
            file_reader.onload = (event) => {
                const filename = "/" + Number(new Date()) + ".json";
                const decoder = new TextDecoder("utf-8")
                const content = decoder.decode(event.target.result);
                paczki_plusplus.FS.writeFile(filename, content);
                paczki_plusplus.FS.chmod(filename, "0777");
                console.log(filename);
                paczki_plusplus.call("loadFile", filename);
            };
            file_reader.readAsArrayBuffer(e.target.files[0]);
        };
        let file_selector = document.createElement("input");
        file_selector.setAttribute("type", "file");
        file_selector.addEventListener("change", (e) => {
            passToCpp(e);
        });
        file_selector.setAttribute("accept", ".json");
        file_selector.click();
    }

    let open = false;
    let alertShowContent = false;
    let alertText = "";
    let alertNonRecoverable = false;
    function showAlert(text, nonRecoverable = false) {
        if (
            (open && !nonRecoverable) ||
            (nonRecoverable && !alertNonRecoverable)
        ) {
            return;
        }
        alertNonRecoverable = nonRecoverable;
        alertText = text;
        alertShowContent = false;
        open = true;
    }
    window.showAlert = showAlert;
</script>

<Dialog
    bind:open
    aria-labelledby="simple-title"
    aria-describedby="simple-content"
    on:SMUIDialog:closed={alertNonRecoverable
        ? () => {
              window.location.reload(true);
          }
        : () => {}}
>
    <!-- Title cannot contain leading whitespace due to mdc-typography-baseline-top() -->
    <Title id="simple-title"
        >{alertNonRecoverable
            ? "Wystąpił nienaprawialny błąd, aplikacja zostanie przeładowana"
            : "Wystąpił niespodziewany błąd"}</Title
    >
    <DialogContent
        >{#if !alertShowContent}<Button
                on:click={() => {
                    alertShowContent = true;
                }}>Wyświetl szczegóły</Button
            >{/if}</DialogContent
    >
    <DialogContent id="simple-content"
        >{#if alertShowContent}{alertText}{/if}</DialogContent
    >
    <Actions>
        <Button>
            <Label>Zamknij</Label>
        </Button>
    </Actions>
</Dialog>

<LayoutGrid>
    <CellLG spanDevices={{ desktop: 6, tablet: 8, phone: 4 }}>
        <Card padded>
            <h3>Podgląd palety</h3>
            <canvas
                bind:this={paczki_plusplus_canvas}
                oncontextmenu="return false"
                id="canvas"
            />
            <p>
                {#if !$pallet_id_list}
                    <Button on:click={() => openFile()} variant="raised">
                        <Label>Otwórz plik</Label>
                    </Button>
                {/if}
                {#if $active_packet}
                    <Button
                        on:click={() =>
                            paczki_plusplus.call(
                                "selectBoxTypeFromBoxPos",
                                null
                            )}
                        variant="raised"
                    >
                        <Label>Pokaż wszystkie paczki tego typu.</Label>
                    </Button>
                    <Button
                        on:click={() =>
                            paczki_plusplus.call("takeBoxOff", null)}
                        variant="raised"
                    >
                        <Label>Zdejmij do schowka.</Label>
                    </Button>
                {/if}
                {#if $pallet_id_list}
                    <Button
                        on:click={() =>
                            paczki_plusplus.call("offerDownload", null)}
                        variant="raised"
                    >
                        <Label>Zapisz plik</Label>
                    </Button>
                {/if}
            </p>

            <FormField>
                <Switch
                    on:SMUISwitch:change={(e) => {
                        $box_pos_order_it = e.detail.selected ? 0 : -1;
                    }}
                />
                <span slot="label">Prezentacja kolejności układania paczek</span
                >
            </FormField>
            <FormField>
                <Switch bind:checked={$color_by_box_pos} />
                <span slot="label">Kolor wg paczki</span>
            </FormField>
            {#if $box_pos_order_it >= 0 && $box_pos_order[0].length - 1 > 0}
                <Slider
                    bind:value
                    on:SMUISlider:input={(e) =>
                        ($box_pos_order_it = e.detail.value)}
                    min={0}
                    max={$box_pos_order[0].length - 1}
                    step={1}
                    discrete
                />
            {/if}
        </Card>
    </CellLG>
    <CellLG spanDevices={{ desktop: 6, tablet: 8, phone: 4 }}>
        <Card padded>
            {#if box_type_val}
                <h3>Podgląd paczki {box_type_val.Id}</h3>
            {:else}
                <h3>Wybierz paczkę z palety lub tabeli poniżej</h3>
            {/if}
            <canvas
                bind:this={packet_viewer_canvas}
                oncontextmenu="return false"
                id="packet_viewer_canvas"
                style="display: {box_type_val ? 'block' : 'none'};"
            />
            {#if box_type_val}
                <ul>
                    <li>
                        Wymiary (x/y/z): {box_type_val.SizeX}/{box_type_val.SizeY}/{box_type_val.SizeZ}
                    </li>
                    <li>Waga: {box_type_val.Weight}</li>
                    <li>
                        Kolor: <span
                            style="color: rgb({box_type_val.ColorR},{box_type_val.ColorG},{box_type_val.ColorB})"
                            >&#9632;</span
                        >
                    </li>
                    <li>
                        Elementy:
                        <ul>
                            {#each Object.entries(items_per_box[box_type_val.$id]) as item, i}
                                <li>
                                    <span>{item[1]}</span>
                                    <span> &times; </span>
                                    <a href={"#item_" + item[0]}>{item[0]}</a
                                    ><span
                                        style="color:rgb({$sku_list[item[0]]
                                            .ColorR}, {$sku_list[item[0]]
                                            .ColorG}, {$sku_list[item[0]]
                                            .ColorB});">&#9632;</span
                                    >
                                </li>
                            {/each}
                        </ul>
                    </li>
                </ul>
            {/if}
        </Card>
    </CellLG>
    <CellLG>
        <Card padded>
            <h3>Schowek</h3>
            Zdjęte paczki:
            <DataTable>
                <Head>
                    <Row>
                        <Cell>Id paczki</Cell>
                        <Cell>Typ paczki</Cell>
                    </Row>
                </Head>
                <Body>
                    {#each Object.values($box_clipboard[0]) as box}
                        <Row>
                            <Cell>{box.$id}</Cell>
                            <Cell
                                ><a
                                    href="javascript:void(0);"
                                    on:click={() =>
                                        ($active_box_type = box.Item1[0].$id)}
                                    >{box.Item1.Id}</a
                                ></Cell
                            >
                            <Cell
                                ><Button
                                    on:click={() =>
                                        paczki_plusplus.call(
                                            "putBoxOn",
                                            box.$id
                                        )}
                                    variant="raised">Połóż</Button
                                ></Cell
                            >
                        </Row>
                    {/each}
                </Body>
            </DataTable>
        </Card>
    </CellLG>
</LayoutGrid>
<!-- 
<br />
<button
    on:click={() => {
        paczki_plusplus.alertMessage();
    }}>Display alert</button
>
{#if $active_packet}
    <h1>Active packet: {$active_packet}</h1>
{/if} -->

<!-- <p>{$active_packet}</p> -->
<!-- {#if $box_type_list}
    <p>{JSON.stringify($box_type_list)}</p>
{/if} -->
<LayoutGrid>
    <CellLG spanDevices={{ desktop: 8, tablet: 8, phone: 4 }}>
        <Card padded>
            <h3>Typy paczek</h3>

            <FormField>
                <Switch bind:checked={pallet_only_box_types} />
                <span slot="label">Tylko paczki z aktywnej palety</span>
            </FormField>
            <DataTable style="width: 100%;">
                <Head>
                    <Row>
                        <Cell>Id</Cell>
                        <Cell>$id</Cell>
                        <!-- <Cell>SkuId</Cell> -->
                        <Cell>Size X</Cell>
                        <Cell>Size Y</Cell>
                        <Cell>Size Y</Cell>
                        <Cell>Weight</Cell>
                        <Cell>Items</Cell>
                    </Row>
                </Head>
                <Body>
                    {#if $box_type_list}
                        {#each box_types as box_type}
                            <Row
                                style={$active_box_type &&
                                box_type.$id == $active_box_type
                                    ? "background-color: lightblue;"
                                    : ""}
                                id={"box_type_" + box_type.$id}
                                on:click={() => {
                                    $active_box_type = box_type.$id;
                                }}
                            >
                                <Cell>{box_type.Id}</Cell>
                                <Cell>{box_type.$id}</Cell>
                                <!-- <Cell>{box_type.SkuId}</Cell> -->
                                <Cell>{box_type.SizeX}</Cell>
                                <Cell>{box_type.SizeY}</Cell>
                                <Cell>{box_type.SizeZ}</Cell>
                                <Cell>{box_type.Weight}</Cell>
                                <Cell over>
                                    {#each Object.entries(items_per_box[box_type.$id]) as item, i}
                                        <span>{item[1]}</span>
                                        <span> &times; </span>
                                        <a href={"#item_" + item[0]}
                                            >{item[0]}</a
                                        >
                                        {#if i !== Object.entries(items_per_box[box_type.$id]).length - 1}
                                            <br />
                                        {/if}
                                    {/each}
                                </Cell>
                            </Row>
                        {/each}
                    {/if}
                </Body>
            </DataTable>
        </Card>
    </CellLG>
    <CellLG spanDevices={{ desktop: 4, tablet: 8, phone: 4 }}>
        <Card padded>
            <h3>Palety</h3>

            <DataTable style="width: 100%;">
                <Head>
                    <Row>
                        <Cell>$id</Cell>
                        <Cell>Boxes</Cell>
                    </Row>
                </Head>
                <Body>
                    {#if $pallet_id_list}
                        {#each $pallet_id_list as pallet}
                            <Row
                                style={$selected_pallet &&
                                pallet[0] == $selected_pallet
                                    ? "background-color: lightblue;"
                                    : ""}
                                id={"pallet_" + pallet[0]}
                                on:click={() => {
                                    $selected_pallet = pallet[0];
                                }}
                            >
                                <Cell>{pallet[0]}</Cell>
                                <Cell over>
                                    {#each Object.entries(pallet[1]) as box_type, i}
                                        <span>{box_type[1]}</span>
                                        <span> &times; </span>
                                        <!-- svelte-ignore a11y-invalid-attribute -->
                                        <a
                                            href="javascript:void(0);"
                                            on:contextmenu={(e) => {
                                                e.preventDefault();
                                                window.location.href =
                                                    "#box_type_" + box_type[0];
                                                return false;
                                            }}
                                            on:click={() => {
                                                $active_box_type = box_type[0];
                                            }}
                                            >{$box_type_list[box_type[0]].Id}</a
                                        >
                                        {#if i !== Object.entries(pallet[1]).length - 1}
                                            <br />
                                        {/if}
                                    {/each}
                                </Cell>
                            </Row>
                        {/each}
                    {/if}
                </Body>
            </DataTable>
        </Card>
    </CellLG>
</LayoutGrid>

<!-- {#if $box_type_list}
    Box types:
    {#each box_type_list_v as box_type}
        box_type {box_type["$id"]} <br />
    {/each}
{/if}

{#if $box_type_list}
    <p>{JSON.stringify($box_type_list)}</p>
{/if}

{#if $box_pos_list}
    <p>{JSON.stringify($box_pos_list)}</p>
{/if}
{#if $pallet_id_list}
    <p>{JSON.stringify($pallet_id_list)}</p>
{/if}
{#if $sku_list}
    <p>{JSON.stringify($sku_list)}</p>
{/if} -->
