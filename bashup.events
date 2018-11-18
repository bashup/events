#!/usr/bin/env bash
event(){ case $1 in error|quote|encode|decode);; *)
	__ev.encode "${2-}";local f n='' e=bashup_event_$REPLY'[1]';f=${e/event/flag}
	case $1 in emit) shift;${!f-};eval "${!e-}"; return ;;on|once|off|has)
		case "${3-}" in @_) n='$#';; @*[^0-9]*);; @[0-9]*) n=$((${3#@}));; esac; ${n:+
		set -- "$1" "$2" "${@:4}" }
		case $1/$# in
			on*/[12]) set -- error "${2-}: missing callback";; */[12]) REPLY=;;
			*) __ev.quote "${@:3}";((${n/\$#/1}))&&REPLY+=' "${@:2:'"$n"'}"';REPLY+=$'\n'
		esac
	esac
esac ;__ev."$@";}
__ev.error(){ echo "$1">&2;return "${2:-64}";}
__ev.quote(){ REPLY=; ${@+printf -v REPLY ' %q' "$@"}; REPLY=${REPLY# };}
__ev.has(){ [[ ${!e-} && $'\n'"${!e}" == *$'\n'"$REPLY"* && ! ${!f-} ]];}
__ev.get(){ ${!f-};REPLY=${!e-};}
__ev.on(){ __ev.has && return;if [[ ! ${!f-} ]];then eval "$e"+='$REPLY';else eval "${!e-};$REPLY";fi;}
__ev.off(){ __ev.has||return 0; n="${!e}"; n=${REPLY:+"${n#"$REPLY"}"}; eval "$e"=$'"${n//\n"$REPLY"/\n}"';[[ ${!e} ]]||unset "${e%\[1]}";}
__ev.fire(){ ${!f-};set -- "$e" "${@:2}"; while [[ ${!1-} ]];do eval "unset ${1%\[1]};${!1}"; done ;}
__ev.all(){ ${!f-};e=${!e-};eval "${e//$'\n'/||return; }";}
__ev.any(){ ${!f-};e=${!e-};eval "${e//$'\n'/&&return|| } ! :";}
__ev.resolve(){
	${!f-};__ev.fire "$@";__ev.quote "$@"
	printf -v n "eval __ev.error 'event \"%s\" already resolved' 70;return" "$1"; eval "${f}"='$n'
	printf -v n 'set -- %s' "$REPLY"; eval "${e}"='$n';readonly "${f%\[1]}" "${e%\[1]}"
}
__ev.resolved(){ [[ ${!f-} ]];}
__ev.once(){ n=${n:-0} n=${n/\$#/_}; event on "$1" "@$n" __ev_once $# "@$n" "$@";}
__ev_once(){ event off "$3" "$2" __ev_once "${@:1:$1+2}"; "${@:4}";}
__ev_jit(){
	local q r=${__ev_jit-} s=$1;((${#r}<250))||__ev_jit=
	while [[ "$s" ]]; do
		r=${s::1};s=${s:1};printf -v q %q "$r";eval 's=${s//'"$q}";printf -v r 'REPLY=${REPLY//%s/_%02x};' "${q/#[~]/[~]}" "'$r";eval "$r";__ev_jit+="$r"
	done
	eval '__ev.encode(){ local LC_ALL=C;REPLY=${1//_/_5f};'\
	"${__ev_jit-}"' [[ $REPLY != *[^_[:alnum:]]* ]] || __ev_jit "${REPLY//[_[:alnum:]]/}";}'
};__ev_jit ''
__ev.decode(){ REPLY=();while (($#));do printf -v n %b "${1//_/\\x}";REPLY+=("$n");shift;done;}
__ev.list(){ eval 'set -- "${!'"${e%\[1]}"'@}"';__ev.decode "${@#bashup_event_}";}