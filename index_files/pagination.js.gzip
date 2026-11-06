const $pagination = function() {
    "use strict";
    
    let current_page = null,
        total_pages = null,
        limit = null;
        
    let dom_pagination = `nav[data-pagination]`,
        dom_pages = `${dom_pagination} div[data-pages]`,
        dom_page = `${dom_pagination} a[data-page]`,
        dom_movers = `${dom_pagination} a[data-mover]`;
    
    /**
     * Sets the pagination array and returns it
     * @returns {string[]} The pagination array
     */
    
    function setPagination() {
        let pages = [],
            has_ellipsis = false;

        if ( current_page < 1 ) {
            current_page = 1;
        }

        if ( current_page > total_pages ) {
            current_page = total_pages;
        }

        for ( let i = 1; i <= total_pages; i++ ) {
            if ( i === 1 || i === total_pages ) {
                pages.push( i );
                has_ellipsis = false;
            }
            else {
                if ( i - limit <= current_page && i + limit >= current_page ) {
                    pages.push( i );
                    has_ellipsis = false;
                }
                else {
                    if ( has_ellipsis === false ) {
                        pages.push( "..." );
                        has_ellipsis = true;
                    }
                }
            }
        }

        return pages;
    }
    
    /**
     * Sets the pagination HTML and returns it as a string
     * @param {Array} pages The pagination array
     * @returns {string} The pagination HTML
     */
    
    function setPaginationTemplate( pages ) {
        let html = [];
        pages.forEach( function( page ) {
            html.push(
                ( current_page == page ) ?
                    `<a href="#" class="is-current-page" data-page="${page}">${page}</a>` :
                    `<a href="#" data-page="${page}">${page}</a>`
            );
        });
        
        return html.join( "" );
    }
    
    /**
     * Sets the pagination HTML, triggers the event, and disables the next or previous button if necessary
     * @param {number} c The current page
     * @param {number} t The total pages
     * @param {number} l The limit
     */
    
    function set( c, t, l ) {
        current_page = parseInt( c );
        total_pages = t || total_pages;
        limit = l || limit;
        
        jQuery( dom_pages ).html(
            setPaginationTemplate(
                setPagination()
            )
        );
        
        jQuery( dom_movers ).each( function() {
            jQuery( this ).removeClass( "is-disabled" );
            
            if ( current_page === 1 && jQuery( this ).attr( "data-mover" ) === "previous" ) {
                jQuery( this ).addClass( "is-disabled" );
            }
            if ( current_page === total_pages && jQuery( this ).attr( "data-mover" ) === "next" ) {
                jQuery( this ).addClass( "is-disabled" );
            }
        });
        
        jQuery( document ).trigger( "pagination:set", {
            current_page: current_page,
            total_pages: total_pages,
            limit: limit
        });
    };
    
    /**
     * When a page number is clicked
     */
    
    function onPageClick() {
        jQuery( document ).off( "click", dom_page ).on( "click", dom_page, function( e ) {
            e.preventDefault();
            set( jQuery( this ).attr( "data-page" ) );
        });
    }
    
    /**
     * When the next or previous button is clicked
     */
    
    function onMoversClick() {
        jQuery( document ).off( "click", dom_movers ).on( "click", dom_movers, function( e ) {
            e.preventDefault();
            
            switch ( jQuery( this ).attr( "data-mover" ) ) {
                case "next": {
                    set( current_page + 1 );
                    break;
                }
                case "previous": {
                    set( current_page - 1 );
                    break;
                }
            }
        });
    }
    
    /**
     * Runs the pagination function
     * @param {number} c The current page
     * @param {number} t The total pages
     * @param {number} l The limit
     */
    
    $pagination.run = function( c, t, l ) {
        set( c, t, l );
        onPageClick();
        onMoversClick();
    };
    
    /**
     * 
     */
    
    $pagination.hide = function() {
        jQuery( dom_pagination ).addClass( "is-hidden" );
    };
    
    /**
     * 
     */
    
    $pagination.show = function() {
        jQuery( dom_pagination ).removeClass( "is-hidden" );
    };
    
    return $pagination;
};
