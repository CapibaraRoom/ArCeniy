# Менеджер модулей **mxz**

**MXZ** - утилита для работы с модулями.

## Исходный скрипт

```
#!/bin/sh

MXZ_CACHE="${MXZ_CACHE:-/var/cache/mxz}"
MXZ_TMP="/tmp/mxz"
ACTIVE_FILE="$MXZ_CACHE/.activemodule"

mkdir -p "$MXZ_CACHE" || { echo "Error: cannot create $MXZ_CACHE" >&2; exit 1; }
mkdir -p "$MXZ_TMP" || { echo "Error: cannot create $MXZ_TMP" >&2; exit 1; }

_escape_path() {
    printf "%s" "$1" | sed 's/[][()&;|<>*?{}$#'"'"'"]/\\&/g; s/ /\\ /g'
}

cmd_help() {
    cat << EOF
Usage: $0 [options] [file]
Options:
  -h            show help
  -am [NAME]    add module config (create) – auto‑generate if NAME omitted
  -dm NAME      delete module config
  -rm OLD NEW   rename module config
  -lm           list all module configs
  -a PATH       append PATH to active module (add line)
  -d PATH       delete PATH from active module (remove matching lines)
  -l [NAME]     list content of module config (active if NAME omitted)
  -c FILE       convertaton (stub)
  -e FILE       extract .xzm module into ./<basename> (without .xzm)
  -p [NAME]     pack module NAME (or active) into ./<NAME>.xzm
  -s [NAME]     select active module (show current if no argument)

If no option is given and a file is provided, action is determined by extension.
EOF
    exit 0
}

_set_active() { echo "$1" > "$ACTIVE_FILE"; }
_get_active() {
    if [ -f "$ACTIVE_FILE" ] && [ -s "$ACTIVE_FILE" ]; then
        cat "$ACTIVE_FILE"
    else
        echo ""
    fi
}

cmd_convert() { echo "convert: $1 (stub)"; }

cmd_extract() {
    module_file="$1"
    [ -z "$module_file" ] && { echo "Error: -e requires a .xzm file" >&2; exit 1; }
    [ ! -f "$module_file" ] && { echo "Error: file '$module_file' not found" >&2; exit 1; }
    case "$module_file" in *.xzm) ;; *) echo "Error: file must have .xzm extension" >&2; exit 1;; esac

    base=$(basename "$module_file" .xzm)
    target_dir="./$base"
    if [ -d "$target_dir" ]; then
        echo "Warning: directory '$target_dir' already exists, overwriting" >&2
        rm -rf "$target_dir"
    fi
    unsquashfs -d "$target_dir" "$module_file"
    echo "Extracted to: $target_dir"
}

cmd_add_module() {
    if [ -n "$1" ]; then
        [ -f "$MXZ_CACHE/$1" ] && { echo "Error: config '$1' already exists" >&2; exit 1; }
        case "$1" in */*) echo "Error: name cannot contain '/'" >&2; exit 1;; esac
        touch "$MXZ_CACHE/$1"
        _set_active "$1"
        echo "Created config: $1 (active)"
    else
        n=1
        while [ -f "$MXZ_CACHE/module-$n" ]; do n=$((n+1)); done
        name="module-$n"
        touch "$MXZ_CACHE/$name"
        _set_active "$name"
        echo "Created config: $name (active)"
    fi
}

cmd_delete_module() {
    [ -z "$1" ] && { echo "Error: -dm requires a config name" >&2; exit 1; }
    [ ! -f "$MXZ_CACHE/$1" ] && { echo "Error: config '$1' does not exist" >&2; exit 1; }
    rm "$MXZ_CACHE/$1"
    if [ "$(_get_active)" = "$1" ]; then
        rm -f "$ACTIVE_FILE"
        echo "Active module cleared"
    fi
    echo "Deleted config: $1"
}

cmd_rename_module() {
    old="$1"; new="$2"
    [ -z "$old" ] && { echo "Error: -rm requires old and new names" >&2; exit 1; }
    [ -z "$new" ] && { echo "Error: -rm requires new name" >&2; exit 1; }
    [ ! -f "$MXZ_CACHE/$old" ] && { echo "Error: config '$old' does not exist" >&2; exit 1; }
    [ -f "$MXZ_CACHE/$new" ] && { echo "Error: config '$new' already exists" >&2; exit 1; }
    case "$new" in */*) echo "Error: name cannot contain '/'" >&2; exit 1;; esac
    mv "$MXZ_CACHE/$old" "$MXZ_CACHE/$new"
    if [ "$(_get_active)" = "$old" ]; then
        _set_active "$new"
        echo "Active module updated to: $new"
    fi
    echo "Renamed config: $old -> $new"
}

cmd_list_modules() {
    files=$(ls -1 "$MXZ_CACHE" 2>/dev/null | grep -v '^\.activemodule$')
    if [ -z "$files" ]; then
        echo "No module configs found"
    else
        echo "Module configs:"
        echo "$files"
    fi
}

cmd_append() {
    path="$1"
    [ -z "$path" ] && { echo "Error: -a requires a path" >&2; exit 1; }
    active="$(_get_active)"
    [ -z "$active" ] && { echo "Error: no active module selected" >&2; exit 1; }
    config="$MXZ_CACHE/$active"
    [ ! -f "$config" ] && { echo "Error: active config '$active' does not exist" >&2; exit 1; }
    echo "$path" >> "$config"
    echo "Appended '$path' to $active"
}

cmd_delete_path() {
    path="$1"
    [ -z "$path" ] && { echo "Error: -d requires a path" >&2; exit 1; }
    active="$(_get_active)"
    [ -z "$active" ] && { echo "Error: no active module selected" >&2; exit 1; }
    config="$MXZ_CACHE/$active"
    [ ! -f "$config" ] && { echo "Error: active config '$active' does not exist" >&2; exit 1; }
    tmpfile=$(mktemp)
    grep -v "^${path}$" "$config" > "$tmpfile"
    mv "$tmpfile" "$config"
    echo "Removed '$path' from $active"
}

cmd_list_content() {
    if [ -n "$1" ]; then
        [ ! -f "$MXZ_CACHE/$1" ] && { echo "Error: config '$1' does not exist" >&2; exit 1; }
        echo "Content of config '$1':"
        cat "$MXZ_CACHE/$1" 2>/dev/null || echo "(empty)"
    else
        active="$(_get_active)"
        if [ -z "$active" ]; then
            echo "No active module, and no name provided" >&2
            exit 1
        fi
        [ ! -f "$MXZ_CACHE/$active" ] && { echo "Error: active config '$active' does not exist" >&2; exit 1; }
        echo "Content of active config '$active':"
        cat "$MXZ_CACHE/$active" 2>/dev/null || echo "(empty)"
    fi
}

_pack_backend() {
    module="$1"
    config="$MXZ_CACHE/$module"
    [ ! -f "$config" ] && { echo "Error: config '$module' not found" >&2; exit 1; }

    pseudo_file="$MXZ_TMP/pseudo_${module}.txt"
    > "$pseudo_file"

    echo "/ d 755 root root" >> "$pseudo_file"

    root_paths=""
    while IFS= read -r raw_path; do
        [ -z "$raw_path" ] && continue
        path=$(echo "$raw_path" | sed 's:/*$::')
        [ -z "$path" ] && continue
        if [ -e "$path" ] || [ -L "$path" ]; then
            root_paths="$root_paths $path"
        else
            echo "Warning: '$path' does not exist, skipping" >&2
        fi
    done < "$config"

    [ -z "$root_paths" ] && { echo "Error: no valid paths to pack" >&2; exit 1; }

    # Родительские директории
    parent_dirs=""
    for path in $root_paths; do
        dir="$path"
        while [ "$dir" != "/" ] && [ "$dir" != "." ] && [ -n "$dir" ]; do
            dir=$(dirname "$dir")
            parent_dirs="$parent_dirs $dir"
        done
    done
    parent_dirs=$(echo "$parent_dirs" | tr ' ' '\n' | sort -u | grep -v '^$' | tr '\n' ' ')
    for dir in $parent_dirs; do
        stat_str=$(stat -L -c "%a %u %g" "$dir" 2>/dev/null)
        if [ -z "$stat_str" ]; then
            mode=755; uid=0; gid=0
        else
            set -- $stat_str
            mode=$1; uid=$2; gid=$3
        fi
        esc_dir="$(_escape_path "$dir")"
        echo "$esc_dir d $mode $uid $gid" >> "$pseudo_file"
    done

    # Обработка каждого корневого пути
    for path in $root_paths; do
        if [ -L "$path" ]; then
            target=$(readlink "$path")
            stat_str=$(stat -c "%a %u %g" "$path" 2>/dev/null)
            if [ -z "$stat_str" ]; then
                mode=777; uid=0; gid=0
            else
                set -- $stat_str
                mode=$1; uid=$2; gid=$3
            fi
            esc_path="$(_escape_path "$path")"
            esc_target="$(_escape_path "$target")"
            echo "$esc_path s $mode $uid $gid $esc_target" >> "$pseudo_file"
        elif [ -f "$path" ]; then
            stat_str=$(stat -L -c "%a %u %g" "$path" 2>/dev/null)
            if [ -z "$stat_str" ]; then
                mode=644; uid=0; gid=0
            else
                set -- $stat_str
                mode=$1; uid=$2; gid=$3
            fi
            esc_path="$(_escape_path "$path")"
            echo "$esc_path f $mode $uid $gid cat $esc_path" >> "$pseudo_file"
        elif [ -d "$path" ]; then
            # Сама директория
            stat_str=$(stat -L -c "%a %u %g" "$path" 2>/dev/null)
            if [ -z "$stat_str" ]; then
                mode=755; uid=0; gid=0
            else
                set -- $stat_str
                mode=$1; uid=$2; gid=$3
            fi
            esc_path="$(_escape_path "$path")"
            echo "$esc_path d $mode $uid $gid" >> "$pseudo_file"

            # Все файлы внутри
            find "$path" -type f -printf "%p\0%m\0%U\0%G\0" 2>/dev/null | while IFS= read -r -d '' file; do
                IFS= read -r -d '' mode
                IFS= read -r -d '' uid
                IFS= read -r -d '' gid
                esc_file="$(_escape_path "$file")"
                echo "$esc_file f $mode $uid $gid cat $esc_file" >> "$pseudo_file"
            done

            # Все ссылки внутри
            find "$path" -type l -printf "%p\0%m\0%U\0%G\0" 2>/dev/null | while IFS= read -r -d '' link; do
                IFS= read -r -d '' mode
                IFS= read -r -d '' uid
                IFS= read -r -d '' gid
                target=$(readlink "$link")
                esc_link="$(_escape_path "$link")"
                esc_target="$(_escape_path "$target")"
                echo "$esc_link s $mode $uid $gid $esc_target" >> "$pseudo_file"
            done

            # Все поддиректории (кроме самой)
            find "$path" -type d -printf "%p\0%m\0%U\0%G\0" 2>/dev/null | while IFS= read -r -d '' dir; do
                IFS= read -r -d '' mode
                IFS= read -r -d '' uid
                IFS= read -r -d '' gid
                [ "$dir" = "$path" ] && continue
                esc_dir="$(_escape_path "$dir")"
                echo "$esc_dir d $mode $uid $gid" >> "$pseudo_file"
            done
        fi
    done

    output="./${module}.xzm"
    mksquashfs - "$output" -noappend -pf "$pseudo_file"
    echo "Module created: $output"
}

cmd_pack() {
    module="$1"
    if [ -z "$module" ]; then
        module="$(_get_active)"
        if [ -z "$module" ]; then
            echo "Error: no module name provided and no active module" >&2
            exit 1
        fi
        echo "Using active module: $module"
    else
        [ ! -f "$MXZ_CACHE/$module" ] && { echo "Error: config '$module' does not exist" >&2; exit 1; }
    fi
    _pack_backend "$module"
}

cmd_select() {
    if [ -n "$1" ]; then
        [ ! -f "$MXZ_CACHE/$1" ] && { echo "Error: config '$1' does not exist" >&2; exit 1; }
        _set_active "$1"
        echo "Active module set to: $1"
    else
        active="$(_get_active)"
        if [ -n "$active" ]; then
            echo "Active module: $active"
        else
            echo "No active module"
        fi
    fi
}

[ $# -eq 0 ] && cmd_help

action=""
args=""
positional=""

while [ $# -gt 0 ]; do
    case "$1" in
        -h|-lm|-am|-dm|-rm|-a|-d|-l|-c|-e|-p|-s)
            action="$1"
            shift
            args=""
            while [ $# -gt 0 ] && [ "${1#-}" = "$1" ]; do
                args="$args $1"
                shift
            done
            args="${args# }"
            ;;
        -*) echo "Unknown option: $1" >&2; exit 1 ;;
        *)  positional="$1"; shift ;;
    esac
done

if [ -z "$action" ] && [ -n "$positional" ]; then
    echo "auto-detect by extension for: $positional (not implemented)"
    exit 0
fi

case "$action" in
    -h) cmd_help ;;
    -lm) cmd_list_modules ;;
    -am) cmd_add_module $args ;;
    -dm) cmd_delete_module $args ;;
    -rm) cmd_rename_module $args ;;
    -a) cmd_append $args ;;
    -d) cmd_delete_path $args ;;
    -l) cmd_list_content $args ;;
    -c) cmd_convert $args ;;
    -e) cmd_extract $args ;;
    -p) cmd_pack $args ;;
    -s) cmd_select $args ;;
    *) cmd_help ;;
esac

exit 0
```
